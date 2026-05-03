Shopify VIP Alerts: AI Summary & Slack Notification for Big Orders

https://n8nworkflows.xyz/workflows/shopify-vip-alerts--ai-summary---slack-notification-for-big-orders-4384


# Shopify VIP Alerts: AI Summary & Slack Notification for Big Orders

### 1. Workflow Overview

This workflow, titled **"Shopify VIP Alerts: AI Summary & Slack Notification for Big Orders"**, is designed to monitor new orders from a Shopify store and send real-time alerts to a Slack channel when high-value orders occur. Specifically, it filters orders with a total price greater than $200, then enriches these events by fetching the customer's historical order data, summarizing it using AI, and delivering a contextualized notification to a dedicated Slack channel.

The workflow is logically divided into the following blocks:

- **1.1 Shopify Order Trigger & Pre-Processing:** Listens for new orders from Shopify, extracts and converts key data such as order total and customer ID, and filters orders based on the $200 threshold.
  
- **1.2 Customer Order History Retrieval & Formatting:** For qualifying orders, fetches the customer’s previous orders from Shopify via API and prepares the data for AI processing by converting arrays to strings.

- **1.3 AI-Powered Summary Generation:** Uses an AI agent (Langchain with OpenAI GPT-4 based model) to generate a concise summary of the customer's purchasing behavior based on their order history.

- **1.4 Slack Notification:** Sends a formatted alert message to a Slack channel, including order details and the AI-generated customer summary.

- **1.5 Low-Value Order Handling:** For orders at or below $200, terminates the workflow early with a no-operation node, avoiding unnecessary processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Shopify Order Trigger & Pre-Processing

**Overview:**  
This block initiates the workflow on new Shopify orders, converts order data types for consistency, and filters orders by value to proceed only with high-value transactions.

**Nodes Involved:**  
- New Shopify Order  
- Type conversion (String to Number)  
- If  
- No Operation, do nothing

**Node Details:**

- **New Shopify Order**  
  - *Type:* Trigger node  
  - *Role:* Listens to Shopify’s `orders/create` webhook for new orders.  
  - *Configuration:* Uses OAuth2 access token authentication.  
  - *Key Data:* Captures full order JSON including `total_price` and `customer.id`.  
  - *Input:* Incoming webhook from Shopify.  
  - *Output:* JSON payload of new order details.  
  - *Edge Cases:* Webhook misconfiguration, authentication failure, delayed webhook delivery.

- **Type conversion (String to Number)**  
  - *Type:* Set node  
  - *Role:* Converts the `total_price` field from string to number to enable numeric comparisons.  
  - *Configuration:* Assigns `total_price` as a number using expression: `={{ $json.total_price }}`.  
  - *Input:* Output from "New Shopify Order".  
  - *Output:* JSON with numeric `total_price`.  
  - *Edge Cases:* Non-numeric or missing `total_price` causing conversion errors.

- **If**  
  - *Type:* Conditional node  
  - *Role:* Evaluates if `total_price` is greater than 200 to branch workflow.  
  - *Configuration:* Condition: `{{ $json.total_price }} > 200` (number comparison).  
  - *Input:* From "Type conversion (String to Number)".  
  - *Output:* Two branches: "true" to proceed, "false" to ignore.  
  - *Edge Cases:* Missing `total_price` field, type mismatches.

- **No Operation, do nothing**  
  - *Type:* NoOp node  
  - *Role:* Terminates the workflow for low-value orders without side effects.  
  - *Input:* From "If" false branch.  
  - *Output:* None.  
  - *Edge Cases:* None, serves as a clean endpoint.

---

#### 1.2 Customer Order History Retrieval & Formatting

**Overview:**  
Triggered for high-value orders, this block fetches the customer’s previous order data from Shopify REST API and prepares it for AI summarization by converting data formats.

**Nodes Involved:**  
- Fetch Customer Order History  
- Type conversion (Array to String)

**Node Details:**

- **Fetch Customer Order History**  
  - *Type:* HTTP Request node  
  - *Role:* Calls Shopify’s API endpoint `/admin/api/2023-07/orders.json?customer_id=...` to retrieve all past orders for the customer.  
  - *Configuration:*  
    - URL constructed dynamically using customer ID from "New Shopify Order": `"https://your-store.myshopify.com/admin/api/2023-07/orders.json?customer_id={{ $('New Shopify Order').item.json.id }}"`  
    - Header includes `X-Shopify-Access-Token` for authentication.  
  - *Input:* From "If" true branch output.  
  - *Output:* JSON containing an array of orders.  
  - *Edge Cases:* API rate limiting, invalid or missing customer ID, token expiration, network failures.

- **Type conversion (Array to String)**  
  - *Type:* Set node  
  - *Role:* Converts the array of orders into a string format suitable for AI input.  
  - *Configuration:* Assigns the field `orders` as a string containing the JSON array of orders: `={{ $json.orders }}`.  
  - *Input:* Output of "Fetch Customer Order History".  
  - *Output:* JSON with stringified `orders`.  
  - *Edge Cases:* Empty order history arrays, malformed JSON data.

---

#### 1.3 AI-Powered Summary Generation

**Overview:**  
This block uses an AI agent to generate a concise summary of the customer's historical buying behavior based on their formatted order data.

**Nodes Involved:**  
- Summarize Order History  
- Ignore Low-Value Order

**Node Details:**

- **Summarize Order History**  
  - *Type:* Langchain Agent node (`@n8n/n8n-nodes-langchain.agent`)  
  - *Role:* Sends a prompt to the AI model to summarize the customer's order history.  
  - *Configuration:*  
    - Prompt text: `"Summarize the customer's order history for Slack. Here is their order data:\n{{ $json.orders }}"`  
    - System message: `"You are a helpful assistant"`  
    - Prompt type: `define` (custom prompt)  
  - *Input:* From "Type conversion (Array to String)".  
  - *Output:* AI-generated summary string in JSON `output` field.  
  - *Edge Cases:* API rate limits, model downtime, malformed input data, timeout errors, token limits.

- **Ignore Low-Value Order**  
  - *Type:* Langchain OpenAI Chat node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)  
  - *Role:* Seems configured but not connected in the main flow; could be legacy or placeholder for low-value order AI processing.  
  - *Configuration:* Model set to `"gpt-4o-mini"`.  
  - *Input:* Not used in main flow.  
  - *Output:* Not used.  
  - *Edge Cases:* Not applicable in active path.

---

#### 1.4 Slack Notification

**Overview:**  
Sends a detailed alert message to a predefined Slack channel containing order details and the AI-generated summary.

**Nodes Involved:**  
- Send Slack Alert

**Node Details:**

- **Send Slack Alert**  
  - *Type:* Slack node  
  - *Role:* Posts a message to Slack channel `C08TTV0CC3E`.  
  - *Configuration:*  
    - Text includes:  
      ```
      🚨 High-Value Order Alert!  
      Customer: {{ first_name }} {{ last_name }}  
      Order Total: ${{ total_price }}  
      Customer Order history Summary: {{ AI summary output }}
      ```  
    - Channel selected from list and set by ID.  
    - OAuth or webhook authentication via configured webhook ID.  
    - No link to workflow included in message.  
  - *Input:* From "Summarize Order History".  
  - *Output:* Confirmation of message sent.  
  - *Edge Cases:* Slack auth errors, channel not found, rate limits, malformed message formatting.

---

### 3. Summary Table

| Node Name                     | Node Type                             | Functional Role                                   | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------------|-------------------------------------|--------------------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| New Shopify Order              | Shopify Trigger                     | Detect new Shopify order                         | (Trigger)                   | Type conversion (String to Number) | 🟢 Shopify Order Trigger & Pre-Processing: detects new orders, extracts key data            |
| Type conversion (String to Number) | Set                              | Converts total_price string to number            | New Shopify Order            | If                              | 🟢 Shopify Order Trigger & Pre-Processing                                                    |
| If                            | If                                 | Checks if order total > 200 to branch workflow   | Type conversion (String to Number) | Fetch Customer Order History (true branch), No Operation (false branch) | 🟢 Shopify Order Trigger & Pre-Processing                                                    |
| Fetch Customer Order History  | HTTP Request                       | Fetches customer's previous orders from Shopify | If (true branch)             | Type conversion (Array to String) | 🧠 Customer Insights via AI: fetches past orders for AI analysis                             |
| Type conversion (Array to String) | Set                              | Converts orders array to string for AI input     | Fetch Customer Order History | Summarize Order History          | 🧠 Customer Insights via AI                                                                 |
| Summarize Order History       | Langchain Agent                    | AI summarizes customer's order history           | Type conversion (Array to String) | Send Slack Alert               | 🧠 Customer Insights via AI                                                                 |
| Send Slack Alert              | Slack                             | Sends alert message to Slack channel              | Summarize Order History      | (Workflow end)                   | 📣 Final Step: Slack Notification                                                           |
| No Operation, do nothing      | NoOp                              | Terminates workflow for low-value orders          | If (false branch)            | (Workflow end)                   | ⛔ Fallback: Low-Value Order                                                                 |
| Ignore Low-Value Order         | Langchain OpenAI Chat             | Unused/placeholder AI node                         | (None active)                | Summarize Order History (not connected) |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **Shopify Trigger** node named `"New Shopify Order"`.  
   - Configure webhook to listen for `orders/create`.  
   - Set authentication method to OAuth2 with valid Shopify API credentials.  
   - Save and activate webhook.

2. **Add Type Conversion (String to Number):**  
   - Add **Set** node named `"Type conversion (String to Number)"`.  
   - Connect input from `"New Shopify Order"`.  
   - Configure to assign a new field `total_price` as a number extracted from `{{$json.total_price}}`.

3. **Add Conditional Node:**  
   - Add **If** node named `"If"`.  
   - Connect input from `"Type conversion (String to Number)"`.  
   - Set condition to check if `total_price > 200`.  
   - Enable version 2.2 of the node for compatibility.

4. **Add No Operation Node:**  
   - Add **No Operation** node named `"No Operation, do nothing"`.  
   - Connect input from the `"If"` node's false branch.

5. **Add HTTP Request for Customer History:**  
   - Add **HTTP Request** node named `"Fetch Customer Order History"`.  
   - Connect input from the `"If"` node's true branch.  
   - Set method to GET.  
   - Configure URL:  
     ```
     https://your-store.myshopify.com/admin/api/2023-07/orders.json?customer_id={{ $('New Shopify Order').item.json.id }}
     ```  
   - Add HTTP header:  
     - `X-Shopify-Access-Token` with your private app or token value.  
   - Use latest API version compatible with your store.

6. **Add Type Conversion (Array to String):**  
   - Add **Set** node named `"Type conversion (Array to String)"`.  
   - Connect input from `"Fetch Customer Order History"`.  
   - Assign field `orders` as a string of `{{$json.orders}}` (stringify the orders array).

7. **Add AI Summarization Node:**  
   - Add **Langchain Agent** node named `"Summarize Order History"`.  
   - Connect input from `"Type conversion (Array to String)"`.  
   - Configure prompt text:  
     ```
     Summarize the customer's order history for Slack. Here is their order data:
     {{ $json.orders }}
     ```  
   - Set system message: `"You are a helpful assistant"`.  
   - Use "define" prompt type.  
   - Configure credentials for OpenAI or compatible AI service.

8. **Add Slack Notification Node:**  
   - Add **Slack** node named `"Send Slack Alert"`.  
   - Connect input from `"Summarize Order History"`.  
   - Configure Slack credentials (OAuth2 or webhook).  
   - Set message text with expressions extracting:  
     - Customer first and last name from `$('New Shopify Order').item.json.billing_address.first_name` and `.last_name`.  
     - Order total from `$('New Shopify Order').item.json.total_price`.  
     - AI summary from current node's output `{{$json.output}}`.  
   - Select the Slack channel ID (e.g., `C08TTV0CC3E`).

9. **Optional:** Add a Langchain OpenAI Chat node named `"Ignore Low-Value Order"` if desired for future AI processing on low-value orders, but it is not connected in this workflow.

10. **Arrange nodes visually and connect as described, ensuring branches from the `"If"` node go to `"Fetch Customer Order History"` on true and `"No Operation"` on false.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                               | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| 🔔 **High-Value Order Alert Workflow Description:** monitors orders >$200, fetches customer history, summarizes with AI, sends Slack alerts to notify team in real-time.                                                                                                                    | Sticky Note in workflow                                                                         |
| 🟢 Shopify Order Trigger & Pre-Processing block focuses on filtering and data extraction early to optimize workflow efficiency.                                                                                                                                                            | Sticky Note1                                                                                   |
| 🧠 Customer Insights via AI block uses OpenAI GPT-4 based Langchain agent to provide actionable customer summaries, improving team responsiveness and personalization.                                                                                                                      | Sticky Note2                                                                                   |
| 📣 Final Slack notification delivers contextualized alerts containing order and AI summary data to a dedicated Slack channel for immediate action.                                                                                                                                          | Sticky Note                                                                                   |
| ⛔ Low-value order fallback avoids unnecessary processing on orders ≤ $200 to maintain workflow performance.                                                                                                                                                                               | Sticky Note                                                                                   |
| Workflow assistance and contact info: For support, contact Yaron@nofluff.online. Additional resources available on YouTube and LinkedIn.                                                                                                                                                   | https://www.youtube.com/@YaronBeen/videos <br> https://www.linkedin.com/in/yaronbeen/          |

---

**Disclaimer:** The provided text exclusively derives from an automated workflow created with n8n, a well-known integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.