Get notified on Gmail, Telegram and Slack on new Stripe purchase

https://n8nworkflows.xyz/workflows/get-notified-on-gmail--telegram-and-slack-on-new-stripe-purchase-3410


# Get notified on Gmail, Telegram and Slack on new Stripe purchase

### 1. Workflow Overview

This workflow is designed to notify users instantly when a customer completes a Stripe checkout session. It is particularly useful for sellers offering multiple products, order bumps, or add-ons in a single purchase. The workflow listens for successful Stripe checkout events, retrieves detailed information about all purchased items in the session, and sends a comprehensive notification to various communication platforms.

**Target Use Cases:**  
- Online sellers and marketers using Stripe Checkout  
- Users who want real-time purchase notifications including detailed product breakdowns  
- Teams preferring notifications via Telegram, WhatsApp, Gmail, Slack, or similar services  

**Logical Blocks:**  
- **1.1 Stripe Event Trigger & Filtering:** Listens for Stripe checkout completion events and filters only paid sessions.  
- **1.2 Retrieve Checkout Line Items:** Fetches all items purchased in the checkout session.  
- **1.3 Retrieve Product Details:** For each line item, fetches detailed product information.  
- **1.4 Aggregate & Format Data:** Aggregates product data into a structured format and prepares notification content.  
- **1.5 Send Notifications:** Sends the formatted purchase details to Telegram, Gmail, and WhatsApp Business Cloud.

---

### 2. Block-by-Block Analysis

#### 1.1 Stripe Event Trigger & Filtering

- **Overview:**  
  This block initiates the workflow by listening to Stripe webhook events for checkout completions and filters to process only paid sessions.

- **Nodes Involved:**  
  - Stripe Trigger  
  - Filter by paid only

- **Node Details:**  

  - **Stripe Trigger**  
    - Type: Stripe Trigger node  
    - Role: Listens for Stripe webhook events, specifically checkout session completions.  
    - Configuration: Uses a webhook ID to receive Stripe events; no additional parameters set.  
    - Inputs: External Stripe webhook calls  
    - Outputs: Emits event data when a checkout session completes  
    - Edge Cases:  
      - Webhook misconfiguration or missing webhook ID may cause no triggers.  
      - Network or Stripe API downtime could delay or prevent event reception.

  - **Filter by paid only**  
    - Type: Filter node  
    - Role: Filters incoming Stripe events to allow only those where the payment status is 'paid'.  
    - Configuration: Expression likely checks `event.data.object.payment_status == 'paid'`.  
    - Inputs: Stripe Trigger output  
    - Outputs: Passes only paid checkout sessions downstream  
    - Edge Cases:  
      - If payment status field is missing or changed in Stripe API, filter may fail.  
      - Unpaid or incomplete sessions are discarded.

#### 1.2 Retrieve Checkout Line Items

- **Overview:**  
  Retrieves all line items (products, bumps, add-ons) from the Stripe checkout session to get a detailed list of purchased items.

- **Nodes Involved:**  
  - Stripe | Get checkout line items  
  - split templates

- **Node Details:**  

  - **Stripe | Get checkout line items**  
    - Type: HTTP Request node  
    - Role: Calls Stripe API to fetch line items for the checkout session ID received from the trigger.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: Stripe API endpoint for checkout session line items (e.g., `/v1/checkout/sessions/{session_id}/line_items`)  
      - Authentication: Uses Stripe credentials configured in n8n  
      - Parameters: Session ID extracted from the trigger event  
    - Inputs: Filter by paid only output (checkout session data)  
    - Outputs: List of line items in the checkout session  
    - Edge Cases:  
      - Invalid or expired session ID may cause API errors.  
      - API rate limits or network issues can cause request failures.

  - **split templates**  
    - Type: Split Out node  
    - Role: Splits the array of line items into individual items for separate processing.  
    - Configuration: Default split on each item in the line items array.  
    - Inputs: Stripe | Get checkout line items output  
    - Outputs: Individual line item objects for further processing  
    - Edge Cases:  
      - Empty line items array results in no output.  
      - Unexpected data structure changes may cause split failure.

#### 1.3 Retrieve Product Details

- **Overview:**  
  For each line item, this block fetches detailed product information from Stripe to enrich the notification content.

- **Nodes Involved:**  
  - Stripe | Get product info  
  - Aggregate

- **Node Details:**  

  - **Stripe | Get product info**  
    - Type: HTTP Request node  
    - Role: Calls Stripe API to retrieve detailed product information for each line item’s product ID.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: Stripe API endpoint for product details (e.g., `/v1/products/{product_id}`)  
      - Authentication: Uses Stripe credentials  
      - Parameters: Product ID extracted from each line item  
    - Inputs: split templates output (individual line items)  
    - Outputs: Detailed product data for each item  
    - Edge Cases:  
      - Missing or invalid product IDs cause API errors.  
      - Network or API rate limits may affect reliability.

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Combines all individual product details back into a single array for formatting.  
    - Configuration: Aggregates all incoming items into one collection.  
    - Inputs: Stripe | Get product info output  
    - Outputs: Aggregated product details array  
    - Edge Cases:  
      - No input items result in empty aggregation.  
      - Large data sets may impact performance.

#### 1.4 Aggregate & Format Data

- **Overview:**  
  This block prepares the final notification message by setting and formatting fields based on aggregated product data.

- **Nodes Involved:**  
  - Set Fields

- **Node Details:**  

  - **Set Fields**  
    - Type: Set node  
    - Role: Constructs and formats the notification message content, including product names, quantities, prices, and customer info.  
    - Configuration:  
      - Uses expressions to build a text or HTML message combining aggregated product data and checkout session details.  
      - May format currency, concatenate strings, and include customer metadata.  
    - Inputs: Aggregate output  
    - Outputs: Formatted notification content ready for sending  
    - Edge Cases:  
      - Missing data fields may cause incomplete messages.  
      - Expression errors if data structure changes.

#### 1.5 Send Notifications

- **Overview:**  
  Sends the formatted notification message to multiple communication channels: Telegram, Gmail, and WhatsApp Business Cloud.

- **Nodes Involved:**  
  - Telegram  
  - Email with HTML design  
  - WhatsApp Business Cloud

- **Node Details:**  

  - **Telegram**  
    - Type: Telegram node  
    - Role: Sends a message to a configured Telegram chat or channel.  
    - Configuration:  
      - Uses Telegram Bot credentials  
      - Message content from Set Fields node  
    - Inputs: Set Fields output  
    - Outputs: Confirmation of message sent  
    - Edge Cases:  
      - Invalid bot token or chat ID causes failure.  
      - Telegram API rate limits or downtime.

  - **Email with HTML design**  
    - Type: Gmail node  
    - Role: Sends an email with the purchase details formatted in HTML.  
    - Configuration:  
      - Uses Gmail OAuth2 credentials  
      - Recipient email address configured  
      - Subject and HTML body from Set Fields content  
    - Inputs: Set Fields output  
    - Outputs: Email send confirmation  
    - Edge Cases:  
      - OAuth token expiration or invalid credentials.  
      - Gmail API quota limits.

  - **WhatsApp Business Cloud**  
    - Type: WhatsApp node  
    - Role: Sends a WhatsApp message via the Business Cloud API.  
    - Configuration:  
      - Uses WhatsApp Business Cloud credentials  
      - Phone number and message content from Set Fields  
    - Inputs: Set Fields output  
    - Outputs: Message send confirmation  
    - Edge Cases:  
      - Invalid phone number or API credentials.  
      - WhatsApp API rate limits or message template restrictions.

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                          | Input Node(s)           | Output Node(s)                          | Sticky Note                                                                                  |
|----------------------------|-----------------------|----------------------------------------|------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| Stripe Trigger             | Stripe Trigger        | Listens for Stripe checkout events     | External Stripe Webhook | Filter by paid only                   |                                                                                              |
| Filter by paid only        | Filter                | Allows only paid checkout sessions     | Stripe Trigger         | Stripe | Get checkout line items      |                                                                                              |
| Stripe | Get checkout line items | HTTP Request          | Fetches all line items from session    | Filter by paid only     | split templates                      |                                                                                              |
| split templates            | Split Out             | Splits line items array into single items | Stripe | Get checkout line items | Stripe | Get product info             |                                                                                              |
| Stripe | Get product info   | HTTP Request          | Retrieves detailed product info        | split templates        | Aggregate                           |                                                                                              |
| Aggregate                 | Aggregate             | Combines product info into one array   | Stripe | Get product info     | Set Fields                         |                                                                                              |
| Set Fields                | Set                   | Formats notification message content   | Aggregate              | Telegram, Email with HTML design, WhatsApp Business Cloud |                                                                                              |
| Telegram                  | Telegram              | Sends notification to Telegram          | Set Fields             | None                              |                                                                                              |
| Email with HTML design    | Gmail                 | Sends notification email with HTML      | Set Fields             | None                              |                                                                                              |
| WhatsApp Business Cloud   | WhatsApp              | Sends notification via WhatsApp Business Cloud | Set Fields             | None                              |                                                                                              |
| Sticky Note3              | Sticky Note           |                                        |                        |                                       |                                                                                              |
| Sticky Note               | Sticky Note           |                                        |                        |                                       |                                                                                              |
| Sticky Note1              | Sticky Note           |                                        |                        |                                       |                                                                                              |
| Sticky Note2              | Sticky Note           |                                        |                        |                                       |                                                                                              |
| Sticky Note4              | Sticky Note           |                                        |                        |                                       |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Stripe Trigger node**  
   - Type: Stripe Trigger  
   - Configure webhook ID (auto-generated or custom) to receive Stripe checkout session events.  
   - No additional parameters needed.

2. **Add Filter node "Filter by paid only"**  
   - Connect Stripe Trigger output to this node.  
   - Set condition to allow only events where `payment_status` equals `'paid'`.

3. **Add HTTP Request node "Stripe | Get checkout line items"**  
   - Connect Filter output to this node.  
   - Configure:  
     - HTTP Method: GET  
     - URL: `https://api.stripe.com/v1/checkout/sessions/{{ $json["data"]["object"]["id"] }}/line_items` (use expression to insert session ID)  
     - Authentication: Stripe API credentials (API key)  
   - No body parameters.

4. **Add Split Out node "split templates"**  
   - Connect output of Stripe | Get checkout line items to this node.  
   - Configure to split the array of line items into individual items.

5. **Add HTTP Request node "Stripe | Get product info"**  
   - Connect output of split templates to this node.  
   - Configure:  
     - HTTP Method: GET  
     - URL: `https://api.stripe.com/v1/products/{{ $json["price"]["product"] }}` (extract product ID from line item)  
     - Authentication: Stripe API credentials.

6. **Add Aggregate node "Aggregate"**  
   - Connect output of Stripe | Get product info to this node.  
   - Configure to aggregate all incoming product info items into a single array.

7. **Add Set node "Set Fields"**  
   - Connect Aggregate output to this node.  
   - Configure fields to build the notification message:  
     - Use expressions to combine product names, quantities, prices, and customer info from the aggregated data and original checkout session.  
     - Format message as plain text or HTML as needed.

8. **Add Telegram node**  
   - Connect Set Fields output to this node.  
   - Configure Telegram Bot credentials.  
   - Set chat ID and message content from Set Fields.

9. **Add Gmail node "Email with HTML design"**  
   - Connect Set Fields output to this node.  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient email, subject, and HTML body from Set Fields.

10. **Add WhatsApp Business Cloud node**  
    - Connect Set Fields output to this node.  
    - Configure WhatsApp Business Cloud credentials.  
    - Set recipient phone number and message content from Set Fields.

11. **Test the workflow**  
    - Trigger a test Stripe checkout session with payment completed.  
    - Verify notifications arrive on Telegram, Gmail, and WhatsApp.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                      |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Using the Stripe API can be tricky, especially with multiple products, order bumps, or add-ons. This workflow solves that by retrieving all items in a checkout session. | Workflow description                                |
| You can receive notifications on Telegram, WhatsApp, Gmail, Slack, Outlook, or any other service by adapting the notification nodes. | Workflow description                                |
| Check out other templates by the creator Solomon at: https://n8n.io/creators/solomon/          | Creator’s template collection link                   |
| The workflow includes detailed formatting to provide a full breakdown of purchased products in notifications. | Workflow description                                |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Get notified on Gmail, Telegram and Slack on new Stripe purchase" workflow. It covers all nodes, their roles, configurations, and potential failure points, enabling advanced users and AI agents to work effectively with this workflow.