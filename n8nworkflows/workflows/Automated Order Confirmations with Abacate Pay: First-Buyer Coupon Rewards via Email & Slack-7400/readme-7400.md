Automated Order Confirmations with Abacate Pay: First-Buyer Coupon Rewards via Email & Slack

https://n8nworkflows.xyz/workflows/automated-order-confirmations-with-abacate-pay--first-buyer-coupon-rewards-via-email---slack-7400


# Automated Order Confirmations with Abacate Pay: First-Buyer Coupon Rewards via Email & Slack

### 1. Workflow Overview

This workflow automates post-purchase communication for transactions processed through the **Abacate Pay** payment gateway. Its primary goal is to send order confirmation emails to customers and notify a Slack channel about new sales. Additionally, it detects if the purchase is the customer's first order. If so, it generates a 10% discount coupon as a reward, which is embedded dynamically in both the email and Slack notification.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Validation:** Receives purchase notifications via webhook and validates access tokens.
- **1.2 Orders Retrieval & First Purchase Check:** Fetches previous orders from Abacate Pay API and determines if the current order is the customer’s first.
- **1.3 Conditional Coupon Generation:** Creates a custom discount coupon if the order is the first for that customer.
- **1.4 Email & Slack Message Construction:** Builds the HTML email body and Slack message blocks, including coupon details if applicable.
- **1.5 Notification Dispatch:** Sends the confirmation email and Slack message.
- **1.6 Error Handling:** Responds with errors when token validation fails.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  Receives incoming webhook requests containing order data and verifies the webhook token for security.

- **Nodes Involved:**  
  - Webhook  
  - Configs  
  - CheckToken  
  - RespondError

- **Node Details:**  

  - **Webhook**  
    - **Type:** Webhook (HTTP POST)  
    - **Configuration:** Listens on a unique path, accepts POST requests with JSON payloads containing order data and a query parameter `webhookSecret` for token validation.  
    - **Input/Output:** Entry point node, outputs full webhook JSON with body and query parameters.  
    - **Edge Cases:** Missing or invalid webhookSecret in query leads to failure.  
    - **Notes:** This node enables external triggers from Abacate Pay’s payment events.

  - **Configs**  
    - **Type:** Set  
    - **Configuration:** Stores static configuration values such as the validation token (`token`: "teste123"), company name, site, and support email.  
    - **Input/Output:** Receives webhook data, outputs JSON with configuration parameters.  
    - **Edge Cases:** Misconfigured token or incorrect company details could cause mismatches downstream.

  - **CheckToken**  
    - **Type:** If  
    - **Configuration:** Validates that the incoming webhook’s `webhookSecret` query parameter matches the stored `token` from Configs. Requires exact, case-sensitive string equality.  
    - **Input/Output:** Takes output from Configs, branches workflow based on token validity.  
    - **Edge Cases:** Token mismatch routes flow to RespondError node. Expression failures if fields are missing.

  - **RespondError**  
    - **Type:** Respond to Webhook  
    - **Configuration:** Returns HTTP 400 status with JSON body `{ "error": "token invalid" }` when token validation fails.  
    - **Input/Output:** Triggered if token check fails. Ends workflow execution with error response.

---

#### 2.2 Orders Retrieval & First Purchase Check

- **Overview:**  
  Fetches all previous orders for customers using Abacate Pay API and determines if the current order is the customer's first purchase.

- **Nodes Involved:**  
  - GetOrders  
  - CheckFirstOrder  
  - If

- **Node Details:**  

  - **GetOrders**  
    - **Type:** HTTP Request  
    - **Configuration:** Makes a GET request to `https://api.abacatepay.com/v1/billing/list` to retrieve all billing orders. Authorization header uses Bearer token (`your_token`) which must be replaced with actual API key.  
    - **Input/Output:** Triggered after successful token validation; outputs JSON list of orders.  
    - **Edge Cases:** API downtime, invalid token, rate limiting, empty or malformed response.

  - **CheckFirstOrder**  
    - **Type:** Code  
    - **Configuration:** JavaScript code compares current webhook order’s customer ID against previous orders. Sets boolean `first` to true if no prior orders exist for that customer.  
    - **Key Expressions:**  
      - `orders.data.filter(order => order.customer?.id === currentOrder.billing.customer.id && order.id !== currentOrder.billing.id)`  
      - Returns `{ first: true/false }`  
    - **Input/Output:** Receives orders from GetOrders and current order from Webhook. Outputs boolean flag `first`.  
    - **Edge Cases:** Missing `customer` or `id` fields, API errors propagating to empty orders, logic errors in filtering.

  - **If**  
    - **Type:** If  
    - **Configuration:** Evaluates if the `first` boolean from CheckFirstOrder is true.  
    - **Input/Output:** Branches flow to coupon creation if true, or skips coupon generation if false.

---

#### 2.3 Conditional Coupon Generation

- **Overview:**  
  If the order is the customer's first, creates a personalized 10% discount coupon via Abacate Pay API.

- **Nodes Involved:**  
  - CreateCustomCoupon

- **Node Details:**  

  - **CreateCustomCoupon**  
    - **Type:** HTTP Request (POST)  
    - **Configuration:**  
      - URL: `https://api.abacatepay.com/v1/coupon/create`  
      - Body JSON includes:  
        - `code`: uppercase customer name with spaces removed, appended `_10` (e.g., `"MATHEUSPEDROSA_10"`)  
        - `notes`: references current order ID  
        - `maxRedeems`: 1  
        - `discountKind`: `"PERCENTAGE"`  
        - `discount`: 10  
        - `metadata`: empty object  
      - Authorization header with Bearer token (`your_token`) required.  
    - **Input/Output:** Triggered only if first order; outputs coupon data including coupon ID.  
    - **Edge Cases:** API errors, duplicate coupon code, invalid token, network issues.

---

#### 2.4 Email & Slack Message Construction

- **Overview:**  
  Constructs a rich HTML email confirming the order and optionally including the coupon. Also builds Slack message blocks for internal notifications.

- **Nodes Involved:**  
  - MakeBodyEmail

- **Node Details:**  

  - **MakeBodyEmail**  
    - **Type:** Code  
    - **Configuration:**  
      - Retrieves order and customer data from Webhook.  
      - Checks if the first order and coupon data exist.  
      - Builds an HTML email with order summary, customer greeting, and coupon section if applicable.  
      - Constructs Slack message blocks including customer info, order ID, payment method, and coupon details if present.  
      - Embeds a button linking to the webhook execution URL for monitoring.  
    - **Key Expressions:** Uses n8n expressions to access node data, e.g., `$('Webhook').item.json.body.data.billing.customer.metadata.email`.  
    - **Input/Output:** Receives coupon data (if any), outputs JSON with `emailHtml` and `slackBlocks`.  
    - **Edge Cases:** Missing customer info, malformed data, coupon API failure leading to missing coupon info, expression evaluation errors.

---

#### 2.5 Notification Dispatch

- **Overview:**  
  Sends the constructed order confirmation email to the customer and posts the notification message to Slack.

- **Nodes Involved:**  
  - Send Email  
  - Slack

- **Node Details:**  

  - **Send Email**  
    - **Type:** Email Send  
    - **Configuration:**  
      - Sends email using SMTP credentials.  
      - To: Customer email from webhook data.  
      - From: Configured company email.  
      - Subject: "Thank You For Your Order!"  
      - HTML Body: Uses `emailHtml` from MakeBodyEmail.  
      - Retries on failure enabled.  
    - **Input/Output:** Triggered after email body construction.  
    - **Edge Cases:** SMTP authentication errors, invalid email addresses, network timeouts.

  - **Slack**  
    - **Type:** Slack Message  
    - **Configuration:**  
      - Sends message to a specific Slack user (`@mth.pedrosa`).  
      - Message type: Block with formatted elements.  
      - Blocks JSON from `slackBlocks` in MakeBodyEmail.  
      - Uses Slack API credentials.  
    - **Input/Output:** Triggered alongside Send Email node.  
    - **Edge Cases:** Slack API rate limits, invalid tokens, user not found.

---

#### 2.6 Error Handling

- **Overview:**  
  Handles invalid webhook token by returning an HTTP 400 error immediately after token check fails.

- **Nodes Involved:**  
  - RespondError

- **Node Details:**  
  - See 2.1 RespondError node details.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                                  | Input Node(s)        | Output Node(s)              | Sticky Note                                                                                             |
|-------------------|---------------------|-------------------------------------------------|----------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| Webhook           | Webhook             | Entry point: Receives order data via POST       | -                    | Configs                     | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| Configs           | Set                 | Stores config vars (token, company info)        | Webhook              | CheckToken                  | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| CheckToken        | If                  | Validates webhook token                          | Configs               | GetOrders, RespondError      | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| RespondError      | Respond to Webhook   | Returns 400 error on invalid token               | CheckToken            | -                           | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| GetOrders         | HTTP Request        | Retrieves previous orders from Abacate Pay API | CheckToken            | CheckFirstOrder             | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| CheckFirstOrder   | Code                | Determines if current order is customer’s first | GetOrders             | If                          | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| If                | If                  | Branches to coupon creation if first order       | CheckFirstOrder       | CreateCustomCoupon, MakeBodyEmail | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| CreateCustomCoupon| HTTP Request        | Creates 10% discount coupon for first-time buyer| If                    | MakeBodyEmail               | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| MakeBodyEmail     | Code                | Builds email HTML and Slack message blocks       | CreateCustomCoupon, If | Slack, Send Email           | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| Slack             | Slack Message       | Sends Slack notification with order details      | MakeBodyEmail         | -                           | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| Send Email        | Email Send          | Sends order confirmation email to customer       | MakeBodyEmail         | -                           | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| Sticky Note       | Sticky Note         | Documentation and author information              | -                    | -                           | ## Abacate Pay to Email & Slack: Automated Confirmation & First-Buyer Coupon (Author info, setup)    |
| Sticky Note1      | Sticky Note         | Displays AbacatePay logo                           | -                    | -                           | ## AbacatePay Logo                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `1b27997e-c86e-4f3d-ac12-61a79400bb1d`)  
   - Purpose: Receive order data with `webhookSecret` query parameter.

2. **Create Configs Node (Set):**  
   - Create variables:  
     - `token`: Your webhook validation token (e.g., "teste123")  
     - `companyName`: Company name (e.g., "N8N")  
     - `companySite`: Company website URL (e.g., "www.n8n.io")  
     - `companyEmail`: Support email (e.g., "your.email@n8n.com")  
   - Connect Webhook → Configs.

3. **Create CheckToken Node (If):**  
   - Condition: Check if `{{ $json.query.webhookSecret }}` equals `{{ $json.token }}` from Configs node.  
   - Connect Configs → CheckToken.  
   - True branch: proceeds to GetOrders.  
   - False branch: goes to RespondError.

4. **Create RespondError Node (Respond to Webhook):**  
   - Response Code: 400  
   - Response Body: `{ "error": "token invalid" }`  
   - Connect CheckToken False → RespondError.

5. **Create GetOrders Node (HTTP Request):**  
   - HTTP Method: GET  
   - URL: `https://api.abacatepay.com/v1/billing/list`  
   - Headers: Authorization: `Bearer your_token` (replace with actual API token)  
   - Connect CheckToken True → GetOrders.

6. **Create CheckFirstOrder Node (Code):**  
   - Code:  
     ```js
     const orders = $('GetOrders').last().json
     const currentOrder = $('Webhook').last().json.body.data
     let first = false
     
     if (!orders.error) {
       let firstOrder = orders.data.filter(order => order.customer?.id === currentOrder.billing.customer.id && order.id !== currentOrder.billing.id)
       if (firstOrder.length === 0) {
         first = true
       }
     }
     return { first }
     ```  
   - Connect GetOrders → CheckFirstOrder.

7. **Create If Node:**  
   - Condition: Check if `{{ $json.first }}` is true.  
   - Connect CheckFirstOrder → If.

8. **Create CreateCustomCoupon Node (HTTP Request):**  
   - HTTP Method: POST  
   - URL: `https://api.abacatepay.com/v1/coupon/create`  
   - Headers: Authorization: `Bearer your_token`  
   - Body (JSON):  
     ```json
     {
       "code": "{{ $('Webhook').item.json.body.data.billing.customer.metadata.name.replaceAll(' ', '').toUpperCase() }}_10",
       "notes": "Coupon sale {{ $('Webhook').item.json.body.data.billing.id }}",
       "maxRedeems": 1,
       "discountKind": "PERCENTAGE",
       "discount": 10,
       "metadata": {}
     }
     ```  
   - Connect If True → CreateCustomCoupon.

9. **Create MakeBodyEmail Node (Code):**  
   - JavaScript that:  
     - Retrieves webhook order and config data.  
     - Checks if first order and coupon data exist.  
     - Builds email HTML with order summary and coupon section (if applicable).  
     - Constructs Slack message blocks with order and coupon info.  
     - Returns JSON with `emailHtml` and `slackBlocks`.  
   - Connect CreateCustomCoupon → MakeBodyEmail (for first order)  
   - Connect If False → MakeBodyEmail (to continue for subsequent orders).

10. **Create Slack Node:**  
    - Type: Slack  
    - User: Select specific Slack user or channel (e.g., `@mth.pedrosa`)  
    - Message Type: Block  
    - Blocks: Use `{{ JSON.stringify($json.slackBlocks) }}` from MakeBodyEmail node output  
    - Connect MakeBodyEmail → Slack.  
    - Configure Slack credentials.

11. **Create Send Email Node:**  
    - Type: Email Send  
    - To Email: `{{ $('Webhook').item.json.body.data.billing.customer.metadata.email }}`  
    - From Email: `{{ $('Configs').item.json.companyEmail }}`  
    - Subject: "Thank You For Your Order!"  
    - HTML Body: `{{ $json.emailHtml }}` from MakeBodyEmail  
    - Enable retry on fail  
    - Connect MakeBodyEmail → Send Email.  
    - Configure SMTP credentials.

12. **Add Sticky Notes:**  
    - Create documentation sticky notes with author, version, description, and branding, placed logically near relevant nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Author: Matheus Pedrosa - https://www.linkedin.com/in/matheus-pedrosa-custodio/                               | Workflow author credit                                                                                       |
| Workflow automates confirmation emails and Slack notifications for Abacate Pay payment events, rewarding first-time buyers with coupons | Workflow description                                                                                        |
| Setup checklist includes configuring API tokens for Abacate Pay nodes and credentials for SMTP and Slack      | Setup instructions                                                                                           |
| Abacate Pay logo image: https://pbs.twimg.com/profile_images/1931057514512019456/6KfF50uJ_400x400.jpg           | Branding for user interface or documentation                                                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.