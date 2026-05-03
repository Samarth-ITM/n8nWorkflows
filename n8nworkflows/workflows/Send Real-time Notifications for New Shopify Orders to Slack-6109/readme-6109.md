Send Real-time Notifications for New Shopify Orders to Slack

https://n8nworkflows.xyz/workflows/send-real-time-notifications-for-new-shopify-orders-to-slack-6109


# Send Real-time Notifications for New Shopify Orders to Slack

### 1. Workflow Overview

This workflow is designed to provide **real-time notifications in Slack** whenever a new order is created in a Shopify store. It targets e-commerce teams who need immediate alerts about new sales to improve order processing speed, coordination, and overall sales management.

The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Captures new order events from Shopify using a webhook trigger.
- **1.2 Data Formatting:** Processes and structures the incoming Shopify order data into a simplified and readable format.
- **1.3 Notification Dispatch:** Sends a formatted message with order details to a specified Slack channel to notify the team instantly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new order creation events on Shopify and triggers the workflow immediately upon such events.

**Nodes Involved:**  
- Shopify Trigger

**Node Details:**

- **Node Name:** Shopify Trigger  
- **Type:** Shopify Trigger node (Webhook listener)  
- **Role:** Listens to Shopify's "orders/create" webhook topic to capture new order events in real time.  
- **Configuration:**  
  - Topic set to `orders/create` to receive notifications when a new order is created.  
  - Uses a webhook ID to establish a webhook endpoint in n8n.  
- **Key Expressions/Variables:** None, as it triggers on external events.  
- **Input Connections:** None (start node).  
- **Output Connections:** Connected to the "Edit Fields" node for data processing.  
- **Version Requirements:** Requires Shopify credentials configured in n8n with webhook permissions.  
- **Edge Cases / Failure Types:**  
  - Shopify API connection issues or revoked webhook permissions.  
  - Network disruptions causing missed webhook calls.  
  - Delay in webhook delivery by Shopify.  
- **Sub-workflow:** None.

---

#### 1.2 Data Formatting

**Overview:**  
This block reformats the raw order data received from Shopify into a simplified JSON object containing only the essential fields needed for the Slack notification.

**Nodes Involved:**  
- Edit Fields (Set node)

**Node Details:**

- **Node Name:** Edit Fields  
- **Type:** Set node (data transformation)  
- **Role:** Extracts and formats key order information for easier consumption downstream.  
- **Configuration:**  
  - Mode: raw JSON output.  
  - Fields set via templated expressions:  
    - `customer_name`: Combines customer's first and last names from the Shopify data.  
    - `order_number`: Shopify order number.  
    - `total_price`: Total price of the order.  
    - `currency`: Currency code for the order.  
    - `items_count`: Number of line items in the order.  
    - `order_url`: Direct URL to the order in Shopify Admin (requires manual update with the store name).  
- **Key Expressions:**  
  ```js
  {
    "customer_name": "{{ $('Shopify Trigger').item.json.customer.first_name }} {{ $('Shopify Trigger').item.json.customer.last_name }}",
    "order_number": "{{ $('Shopify Trigger').item.json.order_number }}",
    "total_price": "{{ $('Shopify Trigger').item.json.total_price }}",
    "currency": "{{ $('Shopify Trigger').item.json.currency }}",
    "items_count": "{{ $('Shopify Trigger').item.json.line_items.length }}",
    "order_url": "https://admin.shopify.com/store/YOUR_STORE/orders/{{ $('Shopify Trigger').item.json.id }}"
  }
  ```  
  Note: Replace `YOUR_STORE` with the actual Shopify store identifier.  
- **Input Connections:** From "Shopify Trigger" node output.  
- **Output Connections:** To "Notify your team" Slack node.  
- **Version Requirements:** Uses n8n version supporting advanced Set node expressions (3.4+).  
- **Edge Cases / Failure Types:**  
  - Missing customer fields (e.g., guest checkouts without customer info) leading to empty or partial customer names.  
  - Incorrect Shopify store URL in the `order_url` field if not updated.  
  - Unexpected data schema changes from Shopify API.  
- **Sub-workflow:** None.

---

#### 1.3 Notification Dispatch

**Overview:**  
This block sends a structured notification message to a designated Slack channel to alert the team about the new order with key details.

**Nodes Involved:**  
- Notify your team (Slack node)

**Node Details:**

- **Node Name:** Notify your team  
- **Type:** Slack node (chat message sender)  
- **Role:** Sends a message to Slack with the formatted order information.  
- **Configuration:**  
  - Authentication via OAuth2 (configured Slack account credentials).  
  - Channel selected from a predefined list, set to channel ID `"C0971EWTRFA"` (usually a channel like `#orders`).  
  - Message text is a JSON object built from the incoming data fields:  
    ```js
    {
      "customer_name": "{{ $json.customer_name }}",
      "order_number": "{{ $json.order_number }}",
      "total_price": "{{ $json.total_price }}",
      "currency": "{{ $json.currency }}",
      "items_count": "{{ $json.items_count }}",
      "order_url": "{{ $json.order_url }}"
    }
    ```  
  - No additional message options set.  
- **Input Connections:** From "Edit Fields" node output.  
- **Output Connections:** None (end node).  
- **Version Requirements:** Uses Slack node version 2.3 or higher for OAuth2 support and channel selection.  
- **Edge Cases / Failure Types:**  
  - Slack authentication token expiry or revocation.  
  - Invalid or missing Slack channel ID.  
  - Slack API rate limits or downtime.  
  - Template evaluation failures if incoming data is incomplete.  
- **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name       | Node Type                 | Functional Role                 | Input Node(s)     | Output Node(s)    | Sticky Note                                                                                                                          |
|-----------------|---------------------------|--------------------------------|-------------------|-------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Shopify Trigger | Shopify Trigger           | Trigger on new Shopify orders  | -                 | Edit Fields       | ## 🛒 SHOPIFY ORDER → SLACK ALERT<br>WORKFLOW PURPOSE:<br>Instantly notify your team in Slack when a new order comes in through Shopify<br><br>SETUP REQUIREMENTS:<br>- Shopify store with webhook access<br>- Slack workspace and channel (#orders recommended)<br>- Shopify app credentials in n8n<br><br>NODES FLOW:<br>1. Shopify Trigger (order/create webhook)<br>2. Edit Fields (format order data)<br>3. Slack (send rich notification) |
| Edit Fields     | Set                       | Format order data              | Shopify Trigger   | Notify your team  | **NODES FLOW**:<br>1. Shopify Trigger (order/create webhook)<br>2. Edit Fields (format order data)<br>3. Slack (send rich notification)  |
| Notify your team | Slack                     | Send Slack notification        | Edit Fields       | -                 | **KEY FEATURES**:<br>✅ Real-time order notifications<br>✅ Customer name & order details<br>✅ Direct link to Shopify admin<br>✅ Order total and item count<br>✅ Professional formatted message<br><br>CONFIGURATION NOTES:<br>- Set Up Shopify Credentials<br>- Customize Slack channel name<br>- Update store URL in order_url field<br>- Test with a sample order<br><br>**BUSINESS VALUE**:<br>⚡ Instant order awareness<br>📈 Faster order processing<br>👥 Team coordination<br>🎯 No missed sales<br><br>TIME TO BUILD: 20 minutes<br>DIFFICULTY: Beginner<br>ROI: High - immediate team efficiency |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Shopify Trigger node:**  
   - Add a new node: `Shopify Trigger`.  
   - Set the `Topic` parameter to `orders/create`.  
   - Ensure you have Shopify credentials configured in n8n with webhook permissions.  
   - This node will receive new order webhooks from Shopify.

2. **Create the Edit Fields node:**  
   - Add a `Set` node named "Edit Fields".  
   - Switch to `Raw JSON` mode for the output.  
   - Set the JSON output with the following template:  
     ```json
     {
       "customer_name": "{{ $('Shopify Trigger').item.json.customer.first_name }} {{ $('Shopify Trigger').item.json.customer.last_name }}",
       "order_number": "{{ $('Shopify Trigger').item.json.order_number }}",
       "total_price": "{{ $('Shopify Trigger').item.json.total_price }}",
       "currency": "{{ $('Shopify Trigger').item.json.currency }}",
       "items_count": "{{ $('Shopify Trigger').item.json.line_items.length }}",
       "order_url": "https://admin.shopify.com/store/YOUR_STORE/orders/{{ $('Shopify Trigger').item.json.id }}"
     }
     ```  
   - Replace `YOUR_STORE` with your actual Shopify store identifier.  
   - Connect the output of "Shopify Trigger" to the input of "Edit Fields".

3. **Create the Slack node:**  
   - Add a `Slack` node, name it "Notify your team".  
   - For Authentication, select OAuth2 credentials for your Slack workspace (set up Slack OAuth2 credentials in n8n beforehand).  
   - Set `Channel` to the desired Slack channel by selecting it from the channel list or entering the channel ID (e.g., `C0971EWTRFA`).  
   - Set the message `Text` to a JSON object using incoming data from the "Edit Fields" node:  
     ```json
     {
       "customer_name": "{{ $json.customer_name }}",
       "order_number": "{{ $json.order_number }}",
       "total_price": "{{ $json.total_price }}",
       "currency": "{{ $json.currency }}",
       "items_count": "{{ $json.items_count }}",
       "order_url": "{{ $json.order_url }}"
     }
     ```
   - Connect the output of "Edit Fields" to this Slack node.

4. **Activate and test the workflow:**  
   - Save and activate the workflow.  
   - Test by creating a sample order in your Shopify store or using the Shopify webhook test interface.  
   - Verify the notification appears correctly in the configured Slack channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                            |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Workflow provides instant team notifications to accelerate order processing and avoid missed sales opportunities.                 | Business value summary from sticky notes                    |
| Customize the Shopify store URL in the `order_url` field in the "Edit Fields" node to ensure direct links work properly.           | Important setup note                                        |
| Slack OAuth2 credentials must be configured in n8n with appropriate permissions to send messages to the desired channel.          | Slack authentication requirement                            |
| Test the workflow with sample Shopify orders to verify data formatting and Slack integration before going live.                    | Recommended best practice                                  |
| Workflow is suitable for beginner users and can be set up in approximately 20 minutes.                                            | Time and difficulty estimate                                |
| For more advanced formatting, consider enhancing the Slack message with blocks or attachments for richer notifications.          | Potential workflow enhancement                              |

---

**Disclaimer:** The provided content is derived exclusively from an automated workflow created with n8n, complying fully with content policies. It contains no illegal, offensive, or protected elements. All processed data is lawful and public.