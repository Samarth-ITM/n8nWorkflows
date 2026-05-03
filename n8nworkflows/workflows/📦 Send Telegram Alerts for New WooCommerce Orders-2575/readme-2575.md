📦 Send Telegram Alerts for New WooCommerce Orders

https://n8nworkflows.xyz/workflows/---send-telegram-alerts-for-new-woocommerce-orders-2575


# 📦 Send Telegram Alerts for New WooCommerce Orders

---

### 1. Workflow Overview

This workflow is designed to send real-time Telegram notifications when a WooCommerce order status changes to "Processing." It is targeted at online store owners who want instant alerts to manage order fulfillment effectively.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Captures incoming order update events from WooCommerce via a webhook.
- **1.2 Order Status Validation:** Checks if the updated order status is "Processing" to filter relevant events.
- **1.3 Message Construction:** Formats the order details into a rich, human-readable Telegram message.
- **1.4 Telegram Notification:** Sends the constructed message to a specified Telegram chat using a bot.

Supporting the workflow are setup instructions and explanatory sticky notes to guide configuration and customization.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives HTTP POST requests from WooCommerce webhooks whenever an order update event occurs.

- **Nodes Involved:**  
  - Receive WooCommerce Order

- **Node Details:**

  - **Receive WooCommerce Order**  
    - Type: Webhook  
    - Role: Entry point to capture WooCommerce order update events.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: A unique webhook path (UUID string) configured in WooCommerce.  
      - No authentication configured here; relies on WooCommerce webhook security.  
    - Inputs: External HTTP POST requests from WooCommerce.  
    - Outputs: JSON payload containing full order data under `$json.body`.  
    - Version: n8n Webhook node version 2.  
    - Edge Cases/Potential Failures:  
      - WooCommerce webhook misconfiguration (wrong URL or inactive webhook).  
      - Payload format changes in WooCommerce API.  
      - Network connectivity issues blocking webhook delivery.

#### 2.2 Order Status Validation

- **Overview:**  
  Filters incoming order data to proceed only if the order status is exactly "processing."

- **Nodes Involved:**  
  - Check if Order Status is Processing

- **Node Details:**

  - **Check if Order Status is Processing**  
    - Type: If  
    - Role: Conditional gate to allow only "processing" status orders to continue.  
    - Configuration:  
      - Condition: `$json.body.status === "processing"` (case-sensitive, strict equality).  
      - Combines conditions with "and" (only one condition here).  
    - Inputs: Output from the webhook node.  
    - Outputs:  
      - `true` branch if condition met (status is "processing").  
      - `false` branch otherwise (stops workflow here).  
    - Version: n8n If node version 2.  
    - Edge Cases/Potential Failures:  
      - Missing or differently named `status` field in payload.  
      - Case mismatch or whitespace errors in status string.  
      - Payload malformed or incomplete.

#### 2.3 Message Construction

- **Overview:**  
  Formats the order data into a structured, styled message suitable for Telegram, including customer info, order details, and product line items.

- **Nodes Involved:**  
  - Design Message Template

- **Node Details:**

  - **Design Message Template**  
    - Type: Code (JavaScript)  
    - Role: Extracts and formats order details into a templated message string with HTML tags for Telegram formatting.  
    - Configuration:  
      - Processes `$json.body` from WooCommerce payload.  
      - Extracts:  
        - Order ID  
        - Customer name (billing first and last)  
        - Total amount (parsed and formatted)  
        - Order creation date/time (GMT, formatted to US locale without timezone conversion)  
        - Billing city and phone  
        - Customer order note (with fallback if empty)  
        - List of ordered products with quantities, formatted as bullet points and bolded quantities.  
      - Returns a single field `orderMessage` containing the final message text.  
    - Inputs: True branch output of the If node.  
    - Outputs: JSON with `orderMessage` string.  
    - Version: n8n Code node version 2.  
    - Edge Cases/Potential Failures:  
      - Missing or null values in billing info or line items.  
      - Unexpected data formats causing runtime JS errors.  
      - Large order data causing message length issues in Telegram.  
      - Date parsing issues if `date_created_gmt` is missing or malformed.

#### 2.4 Telegram Notification

- **Overview:**  
  Sends the formatted message to a Telegram chat using a configured Telegram bot.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - Type: Telegram  
    - Role: Sends a message to a Telegram chat ID using Telegram Bot API.  
    - Configuration:  
      - Text: Uses `{{ $json.orderMessage }}` expression from previous node.  
      - Chat ID: Placeholder `<Your-Chat-ID>` to be replaced with actual chat ID.  
      - Additional fields:  
        - Parse Mode: HTML (to render bold tags and emojis).  
        - Append Attribution: true (appends "via n8n" attribution).  
      - Telegram credentials required: Telegram Bot token configured in n8n credentials.  
    - Inputs: Output of the "Design Message Template" node.  
    - Outputs: Telegram API response data.  
    - Version: Telegram node version 1.2.  
    - Edge Cases/Potential Failures:  
      - Invalid Telegram bot token or expired credentials.  
      - Incorrect or missing chat ID.  
      - Telegram API rate limits or message length limits.  
      - Network or API errors.

#### Supporting Nodes: Sticky Notes

- **Sticky Note** (large, detailed setup guide)  
  - Provides step-by-step instructions on:  
    - WooCommerce webhook setup  
    - Telegram bot creation  
    - n8n Telegram credentials setup  
    - Node configuration tips  
    - How to activate and test the workflow  
  - Contains tips and troubleshooting notes.

- **Sticky Note1** (workflow description)  
  - Summarizes workflow purpose and target users.

---

### 3. Summary Table

| Node Name                        | Node Type         | Functional Role                      | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                                                                                                                                     |
|---------------------------------|-------------------|------------------------------------|-----------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Receive WooCommerce Order        | Webhook           | Receives WooCommerce order updates | (external HTTP POST)         | Check if Order Status is Processing |                                                                                                                                                                                                                                 |
| Check if Order Status is Processing | If                | Filters orders with status "processing" | Receive WooCommerce Order    | Design Message Template    |                                                                                                                                                                                                                                 |
| Design Message Template          | Code              | Formats order details message       | Check if Order Status is Processing | Telegram                  |                                                                                                                                                                                                                                 |
| Telegram                        | Telegram          | Sends message to Telegram chat      | Design Message Template      | (end)                     |                                                                                                                                                                                                                                 |
| Sticky Note                     | Sticky Note       | Setup instructions and tips         | N/A                         | N/A                       | Contains comprehensive setup instructions for WooCommerce webhook, Telegram bot creation, credential setup, node configuration, activation, and testing.                                                                       |
| Sticky Note1                    | Sticky Note       | Workflow purpose description        | N/A                         | N/A                       | Summarizes workflow function and target users.                                                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it**: "Send Telegram Alerts for New WooCommerce Orders".

2. **Add a Webhook node**:  
   - Name: `Receive WooCommerce Order`  
   - Set HTTP Method: `POST`  
   - Set Path: Use a unique identifier (e.g., a UUID string) that will be configured in WooCommerce webhook.  
   - Leave authentication off unless you add custom webhook security.  
   - Save.

3. **Add an If node**:  
   - Name: `Check if Order Status is Processing`  
   - Connect the output of the webhook node to this If node.  
   - Configure the condition (Version 2 recommended):  
     - Condition type: String  
     - Left Value: Expression — `{{$json.body.status}}`  
     - Operator: Equals  
     - Right Value: `processing` (case-sensitive)  
   - This node branches the workflow: only proceed if order status is "processing".

4. **Add a Code node**:  
   - Name: `Design Message Template`  
   - Connect the `true` output of the If node to this node.  
   - Paste the following JavaScript code (adjust if necessary):
     ```javascript
     const lineItems = $json.body.line_items;
     const totalAmount = parseInt($json.body.total).toLocaleString();

     const filteredItems = lineItems.map(item => {
       const name = item.name;
       const quantity = item.quantity;
       return `🔹 ${name}<b> (${quantity} items)</b>`;
     }).join('\n');

     let dateCreated = new Date($json.body.date_created_gmt || new Date());

     let formattedDate = dateCreated.toLocaleString('en-US', {
       year: 'numeric',
       month: 'long',
       day: 'numeric',
       hour: '2-digit',
       minute: '2-digit',
       hour12: false
     });

     const orderInfo = `
     🆔 <b>Order ID:</b> ${$json.body.id}

     👦🏻 <b>Customer Name:</b> ${$json.body.billing.first_name} ${$json.body.billing.last_name}

     💵 <b>Amount:</b> ${totalAmount}

     📅 <b>Order Date:</b>
     ➖ ${formattedDate}

     🏙 <b>City:</b> ${$json.body.billing.city}

     📞 <b>Phone:</b> ${$json.body.billing.phone}

     ✍🏻 <b>Order Note:</b>
     ${$json.body.customer_note || 'No notes'}

     📦 <b>Ordered Products:</b>

     ${filteredItems}
     `;

     return {
       orderMessage: orderInfo.trim()
     };
     ```
   - Save.

5. **Add a Telegram node**:  
   - Name: `Telegram`  
   - Connect the output of the Code node to this node.  
   - Configure:  
     - Credentials: Select or create Telegram Bot credentials with your Bot API token.  
     - Chat ID: Enter the Telegram chat ID where notifications will be sent. Use @userinfobot on Telegram to find this ID.  
     - Text: Use expression `{{ $json.orderMessage }}` to send the formatted message.  
     - Additional Fields:  
       - Parse mode: `HTML` (to support bold and emojis)  
       - Append Attribution: `true`  
   - Save.

6. **Configure WooCommerce Webhook**:  
   - In WooCommerce admin panel, navigate to **WooCommerce > Settings > Advanced > Webhooks**.  
   - Add a new webhook:  
     - Status: Active  
     - Topic: Order updated  
     - Delivery URL: Use the URL generated by the `Receive WooCommerce Order` webhook node.  
     - Save.

7. **Activate the workflow in n8n**:  
   - Ensure all nodes are connected properly.  
   - Switch workflow state to active.

8. **Testing**:  
   - Place a new order in WooCommerce.  
   - Change order status to "Processing".  
   - Verify Telegram receives the notification with the formatted order message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Contact the workflow author on Telegram for support or customizations: https://t.me/amir676080                                                                                                                                          | Telegram contact link                          |
| Customize the message format by editing the "Design Message Template" node’s JavaScript code to include additional order or customer details as required.                                                                              | Workflow customization                        |
| Use @userinfobot in Telegram to find your personal or group chat ID for sending notifications.                                                                                                                                           | Telegram Chat ID retrieval                      |
| The workflow uses HTML parse mode for Telegram messages to support rich formatting such as bold text and emojis.                                                                                                                       | Telegram API message formatting                |
| WooCommerce webhook must be configured to fire on "Order updated" events and be active for this workflow to trigger correctly.                                                                                                          | WooCommerce webhook setup                       |
| This workflow is designed for n8n version supporting Webhook node v2, If node v2, and Telegram node v1.2 or higher for best compatibility.                                                                                              | Version compatibility                           |

---

This document provides a full understanding, node-by-node analysis, and instructions to recreate and customize the "Send Telegram Alerts for New WooCommerce Orders" workflow in n8n. It covers error possibilities, data flow, and integration points for efficient maintenance and extension.