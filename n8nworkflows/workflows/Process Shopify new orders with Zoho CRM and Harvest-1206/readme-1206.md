Process Shopify new orders with Zoho CRM and Harvest

https://n8nworkflows.xyz/workflows/process-shopify-new-orders-with-zoho-crm-and-harvest-1206


# Process Shopify new orders with Zoho CRM and Harvest

### 1. Workflow Overview

This workflow automates processing new Shopify orders by integrating with Zoho CRM, Harvest invoicing, Trello task management, Gmail for emailing customers, and MailChimp for marketing campaigns. It is triggered by the creation of new orders in Shopify and performs the following logical blocks:

- **1.1 Input Reception:** Detect new Shopify orders using a webhook trigger and normalize order/customer data.
- **1.2 CRM Integration:** Upsert customer contact information in Zoho CRM.
- **1.3 Invoicing and Task Management:** Create an invoice in Harvest from the order details and create a Trello card linked to the invoice.
- **1.4 Conditional Customer Communication:** Based on order value, send either a thank-you email or a coupon email via Gmail.
- **1.5 Marketing Campaign Enrollment:** If order value exceeds 50, add the customer to a MailChimp campaign tagged for high-value customers.

The workflow includes key conditional branching and uses multiple API credentials requiring prior setup (Shopify, Zoho, Harvest, Trello, Gmail, MailChimp). It also requires customization of Trello List ID and Harvest Account ID.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new Shopify order is created and extracts key customer and order fields for downstream processing.

- **Nodes Involved:**  
  - `order created` (Shopify Trigger)  
  - `Set fields` (Set node)

- **Node Details:**

  - **`order created`**  
    - *Type:* Shopify Trigger  
    - *Role:* Listens for new order creation events on Shopify via webhook.  
    - *Configuration:* Trigger on topic `orders/create`. Requires Shopify API credentials.  
    - *Inputs:* External webhook event.  
    - *Outputs:* Raw order JSON data.  
    - *Edge Cases:* Webhook connectivity failures, API rate limits, missing order fields.  
    - *Notes:* Webhook ID is configured (`qwertz`).

  - **`Set fields`**  
    - *Type:* Set  
    - *Role:* Extracts and normalizes customer and order data into simplified fields for later nodes.  
    - *Configuration:* Maps nested JSON fields like customer first name, last name, email, phone, shipping address, and total order value to flat variables (e.g., `customer_firstname`, `order_value`).  
    - *Key Expressions:* Uses expressions in double curly braces to access nested JSON from Shopify order.  
    - *Inputs:* Output from `order created`.  
    - *Outputs:* Cleaned JSON with selected fields only.  
    - *Edge Cases:* Missing or malformed customer or shipping fields can cause expression errors.

#### 2.2 CRM Integration

- **Overview:**  
  Updates or creates customer contact in Zoho CRM using the extracted customer data.

- **Nodes Involved:**  
  - `Zoho` (Zoho CRM node)

- **Node Details:**

  - **`Zoho`**  
    - *Type:* Zoho CRM  
    - *Role:* Upserts contact records based on customer last name, with additional fields for email, phone, first name, and mailing address.  
    - *Configuration:* Resource `contact`, operation `upsert`. Mailing address fields are mapped from normalized data. Uses OAuth2 credentials named `zoho_api`.  
    - *Inputs:* From `Set fields`.  
    - *Outputs:* Updated or created contact record info.  
    - *Edge Cases:* OAuth token expiration, API rate limits, missing mandatory fields in Zoho, upsert conflicts.

#### 2.3 Invoicing and Task Management

- **Overview:**  
  Creates an invoice in Harvest for the order and then creates a corresponding Trello card for task tracking.

- **Nodes Involved:**  
  - `Harvest` (Harvest node)  
  - `Trello` (Trello node)

- **Node Details:**

  - **`Harvest`**  
    - *Type:* Harvest  
    - *Role:* Creates an invoice resource using order details.  
    - *Configuration:* Account ID must be replaced with user’s own (`12345` is placeholder). Sets currency, issue date, payment terms, and purchase order number based on Shopify order data. Uses Harvest access token credentials `harvest_token`.  
    - *Inputs:* From `Set fields`.  
    - *Outputs:* Invoice creation confirmation and ID.  
    - *Edge Cases:* Invalid account ID, expired token, API limits, missing invoice fields.

  - **`Trello`**  
    - *Type:* Trello  
    - *Role:* Creates a Trello card named after the Shopify order number in a specified Trello list to track the order’s invoice.  
    - *Configuration:* Requires user to replace `listId` placeholder `list01` with their Trello list ID. Card name is dynamic using order number. URL source is the Shopify order status URL. Uses Trello API credentials `trello_nodeqa`.  
    - *Inputs:* From `Harvest`.  
    - *Outputs:* Created Trello card info.  
    - *Edge Cases:* Invalid Trello list ID, permission errors, API rate limits.  
    - *Notes:* See [Trello node docs](https://docs.n8n.io/nodes/n8n-nodes-base.trello/#example-usage) for list ID setup.

#### 2.4 Conditional Customer Communication

- **Overview:**  
  Branches workflow based on whether the order value is above 50. Sends either a discount coupon email plus MailChimp subscription or a simple thank-you email.

- **Nodes Involved:**  
  - `IF` (If node)  
  - `Gmail - coupon` (Gmail node)  
  - `Gmail - thankyou` (Gmail node)

- **Node Details:**

  - **`IF`**  
    - *Type:* If  
    - *Role:* Evaluates if the `order_value` is larger than 50.  
    - *Configuration:* Numeric comparison, `order_value > 50`.  
    - *Inputs:* From `Set fields`.  
    - *Outputs:* Two branches — true (order_value > 50) and false.  
    - *Edge Cases:* Missing or non-numeric order_value, causing expression failure.

  - **`Gmail - coupon`**  
    - *Type:* Gmail  
    - *Role:* Sends an email containing a 15% discount coupon.  
    - *Configuration:* Email recipient is customer email; subject and body contain a personalized coupon message using customer first name. Uses Gmail OAuth2 credentials `gmail`.  
    - *Inputs:* True branch of `IF`.  
    - *Outputs:* Email sent confirmation.  
    - *Edge Cases:* Gmail API limits, OAuth token expiration, invalid email address.

  - **`Gmail - thankyou`**  
    - *Type:* Gmail  
    - *Role:* Sends a thank-you email without coupon.  
    - *Configuration:* Similar to coupon email but with a thank-you message only. Uses Gmail OAuth2 credentials `gmail`.  
    - *Inputs:* False branch of `IF`.  
    - *Outputs:* Email sent confirmation.  
    - *Edge Cases:* Same as coupon email.

#### 2.5 Marketing Campaign Enrollment

- **Overview:**  
  Adds customers with orders over 50 to a MailChimp campaign with a “high-order” tag.

- **Nodes Involved:**  
  - `Mailchimp` (MailChimp node)

- **Node Details:**

  - **`Mailchimp`**  
    - *Type:* MailChimp  
    - *Role:* Tags the customer email in a specified MailChimp list to enroll them in a targeted campaign.  
    - *Configuration:* List ID `qwertz` (should be replaced accordingly). Tags: `high-order`. Email is customer email from `Set fields`. Uses MailChimp API credentials `mailchimp_API`.  
    - *Inputs:* From `Gmail - coupon` node (true branch only).  
    - *Outputs:* Confirmation of member tag addition.  
    - *Edge Cases:* Invalid list ID, API authentication failure, email format errors.

---

### 3. Summary Table

| Node Name       | Node Type           | Functional Role                      | Input Node(s)     | Output Node(s)              | Sticky Note                                                                                                  |
|-----------------|---------------------|------------------------------------|-------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| order created   | Shopify Trigger     | Trigger workflow on new Shopify order | Webhook external  | Set fields                 |                                                                                                              |
| Set fields      | Set                 | Normalize and flatten order/customer data | order created     | Harvest, IF, Zoho          |                                                                                                              |
| Zoho            | Zoho CRM             | Upsert customer contact in Zoho CRM | Set fields        |                            |                                                                                                              |
| Harvest         | Harvest              | Create invoice from order data      | Set fields        | Trello                     | Replace *Account ID* in Harvest node with your own (see https://docs.n8n.io/credentials/harvest/#using-access-token) |
| Trello          | Trello               | Create Trello card for order invoice | Harvest           |                            | Replace *List ID* in Trello node with your own (see https://docs.n8n.io/nodes/n8n-nodes-base.trello/#example-usage)  |
| IF              | If                   | Branch by order value > 50          | Set fields        | Gmail - coupon, Gmail - thankyou |                                                                                                              |
| Gmail - coupon  | Gmail                | Send coupon email for orders > 50   | IF (true branch)  | Mailchimp                  |                                                                                                              |
| Gmail - thankyou| Gmail                | Send thank-you email for orders ≤ 50 | IF (false branch) |                            |                                                                                                              |
| Mailchimp       | MailChimp            | Add high-value customers to campaign | Gmail - coupon    |                            |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger node (`order created`):**  
   - Type: Shopify Trigger  
   - Credentials: Setup Shopify API credentials (`shopify_nodeqa`).  
   - Parameters: Set Topic to `orders/create`.  
   - Save webhook ID if needed.

2. **Add Set node (`Set fields`):**  
   - Connect from `order created`.  
   - Configure to extract and flatten these fields using expressions:  
     - `customer_phone` = `{{$json["customer"]["default_address"]["phone"]}}`  
     - `customer_zipcode` = `{{$json["shipping_address"]["zip"]}}`  
     - `order_value` = `{{$json["current_total_price"]}}`  
     - `customer_firstname` = `{{$json["customer"]["first_name"]}}`  
     - `customer_lastname` = `{{$json["customer"]["last_name"]}}`  
     - `customer_email` = `{{$json["customer"]["email"]}}`  
     - `customer_country` = `{{$json["shipping_address"]["country"]}}`  
     - `customer_street` = `{{$json["shipping_address"]["address1"]}}`  
     - `customer_city` = `{{$json["shipping_address"]["city"]}}`  
     - `customer_province` = `{{$json["shipping_address"]["province"]}}`  
   - Keep only set fields.

3. **Create Zoho CRM node (`Zoho`):**  
   - Connect from `Set fields`.  
   - Credentials: Setup Zoho OAuth2 credentials (`zoho_api`).  
   - Resource: `contact`, Operation: `upsert`.  
   - Set last name (`{{$json["customer_lastname"]}}`).  
   - Additional fields: First Name, Email, Phone, Mailing Address (street, city, state, zip, country) from corresponding variables.  
   - Fill mailing state's field as empty string `""`.

4. **Create Harvest node (`Harvest`):**  
   - Connect from `Set fields`.  
   - Credentials: Setup Harvest API credentials (`harvest_token`).  
   - Operation: `create` invoice.  
   - Parameters:  
     - Client ID: Provide your client identifier (e.g., `shopify_client`).  
     - Account ID: Replace placeholder `12345` with your Harvest account ID.  
     - Additional fields:  
       - Currency: `{{$node["order created"].json["currency"]}}`  
       - Issue date: `{{$node["order created"].json["processed_at"]}}`  
       - Payment term: `net 15`  
       - Purchase order: `{{$node["order created"].json["order_number"]}}`

5. **Create Trello node (`Trello`):**  
   - Connect from `Harvest`.  
   - Credentials: Setup Trello API credentials (`trello_nodeqa`).  
   - Parameters:  
     - Name: `"Shopify order {{$node["order created"].json["order_number"]}}"`  
     - List ID: Replace placeholder `list01` with your Trello list ID.  
     - URL Source: `{{$node["order created"].json["order_status_url"]}}`

6. **Create If node (`IF`):**  
   - Connect from `Set fields`.  
   - Condition: Numeric, `order_value` > 50.

7. **Create Gmail node for coupon email (`Gmail - coupon`):**  
   - Connect from `IF` true branch.  
   - Credentials: Setup Gmail OAuth2 (`gmail`).  
   - To: `{{$node["Set fields"].json["customer_email"]}}`  
   - Subject: `"Your Shopify order"`  
   - Message:  
     ```
     Hi {{$json["customer_firstname"]}},

     Thank you for your order! Here's a 15% coupon code to use for your next order: COUPON15

     Best,
     Shop Owner
     ```

8. **Create MailChimp node (`Mailchimp`):**  
   - Connect from `Gmail - coupon`.  
   - Credentials: Setup MailChimp API credentials (`mailchimp_API`).  
   - Parameters:  
     - List: Replace with your MailChimp list ID (`qwertz` is placeholder).  
     - Email: `{{$node["Set fields"].json["customer_email"]}}`  
     - Tags: Add tag `"high-order"`.

9. **Create Gmail node for thank-you email (`Gmail - thankyou`):**  
   - Connect from `IF` false branch.  
   - Credentials: Same Gmail OAuth2 (`gmail`).  
   - To: `{{$node["Set fields"].json["customer_email"]}}`  
   - Subject: `"Your Shopify order"`  
   - Message:  
     ```
     Hi {{$node["Set fields"].json["customer_firstname"]}},
     Thank you for your order! We're getting it ready for shipping it to you.

     Best,
     Shop Owner
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Replace *List ID* in Trello node with your own ID as per instructions in n8n docs.              | https://docs.n8n.io/nodes/n8n-nodes-base.trello/#example-usage                                                 |
| Replace *Account ID* in Harvest node with your own account ID.                                  | https://docs.n8n.io/credentials/harvest/#using-access-token                                                    |
| Ensure all API credentials (Shopify, Zoho, Harvest, Trello, Gmail, MailChimp) are properly set up with OAuth2 or API token as required. | n8n official docs for each respective node.                                                                   |
| The workflow assumes order JSON structure from Shopify webhook matches expected fields exactly; changes in Shopify API may require node expression updates. | Shopify API docs: https://shopify.dev/api                                                                                |

---

This document enables developers and automation engineers to fully understand, reproduce, and maintain this multi-service order processing workflow with clear instructions and insights on potential failure points.