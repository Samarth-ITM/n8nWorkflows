Process Ko-fi Donations, Subscriptions & Shop Orders with Webhook Verification

https://n8nworkflows.xyz/workflows/process-ko-fi-donations--subscriptions---shop-orders-with-webhook-verification-3186


# Process Ko-fi Donations, Subscriptions & Shop Orders with Webhook Verification

### 1. Workflow Overview

This workflow automates the processing of payment notifications from Ko-fi, a platform used by creators to receive donations, subscriptions, and shop orders. It is designed for content creators, artists, and developers who want to streamline the management of incoming financial support by automatically verifying, categorizing, and preparing payment data for further actions.

The workflow is logically divided into the following blocks:

- **1.1 Webhook Reception**: Listens for incoming POST requests from Ko-fi containing payment notifications.
- **1.2 Data Preparation and Verification**: Extracts the relevant data from the webhook payload and verifies the authenticity of the request using a pre-configured verification token.
- **1.3 Payment Type Differentiation**: Determines the type of payment received (Donation, Subscription, or Shop Order) and routes the data accordingly.
- **1.4 Payment Type Handling**: For each payment type, prepares a structured data set with relevant fields for downstream processing.
- **1.5 Subscription Specific Check**: For subscriptions, checks if the payment is the first subscription payment to enable conditional handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Reception

- **Overview:**  
  This block receives incoming webhook POST requests from Ko-fi, serving as the entry point for all payment notifications.

- **Nodes Involved:**  
  - `Webhook`

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook node (HTTP POST listener)  
    - Configuration: Listens on a unique path (`83f4e1de-2011-487c-a9f7-be6ccbac0782`) for POST requests. No additional options configured.  
    - Inputs: External HTTP POST requests from Ko-fi.  
    - Outputs: Passes raw webhook data downstream.  
    - Edge Cases:  
      - Incoming requests not from Ko-fi (mitigated by token verification later).  
      - HTTP method other than POST (ignored by configuration).  
    - Sticky Note: Instructions to set the webhook URL in Ko-fi’s webhook management page.  
    - Link: [Ko-fi Webhooks Management](https://ko-fi.com/manage/webhooks)

#### 2.2 Data Preparation and Verification

- **Overview:**  
  Extracts the actual payment data from the webhook payload and verifies the request’s authenticity by comparing the verification token sent by Ko-fi with the token stored in the workflow.

- **Nodes Involved:**  
  - `Prepare`  
  - `Check token`  
  - `Stop and Error`

- **Node Details:**  
  - **Prepare**  
    - Type: Set node  
    - Configuration:  
      - Sets a static `verificationToken` string (user must input their Ko-fi verification token here).  
      - Extracts the `data` object from the webhook payload (`$json.body.data`) and assigns it to a new field `body` for easier access downstream.  
    - Inputs: Raw webhook data from `Webhook` node.  
    - Outputs: Prepared data with verification token and cleaned body.  
    - Edge Cases:  
      - Missing or malformed `data` object in webhook payload.  
    - Sticky Note: Instructions to set the verification token here with a link to Ko-fi settings.  
    - Link: [Ko-fi Webhooks Management](https://ko-fi.com/manage/webhooks)  

  - **Check token**  
    - Type: If node  
    - Configuration:  
      - Compares the `verification_token` field from the incoming payload (`$json.body.verification_token`) with the stored `verificationToken`.  
      - If they match, passes data to the next node; otherwise, triggers error handling.  
    - Inputs: Prepared data from `Prepare` node.  
    - Outputs:  
      - True branch: proceeds to payment type checking.  
      - False branch: triggers `Stop and Error`.  
    - Edge Cases:  
      - Token mismatch (possible spoofed or invalid request).  
      - Missing token field in payload.  

  - **Stop and Error**  
    - Type: Stop and Error node  
    - Configuration:  
      - Stops workflow execution and returns an error message "Invalid verification token".  
    - Inputs: From `Check token` node’s false branch.  
    - Outputs: None (workflow halts).  
    - Edge Cases: None beyond token mismatch.

#### 2.3 Payment Type Differentiation

- **Overview:**  
  Determines the payment type from the verified webhook data and routes the workflow to the appropriate processing node.

- **Nodes Involved:**  
  - `Check type`

- **Node Details:**  
  - **Check type**  
    - Type: Switch node  
    - Configuration:  
      - Checks the `type` field in the webhook payload (`$json.body.type`).  
      - Routes to one of three branches based on value: "Donation", "Subscription", or "Shop Order".  
    - Inputs: Verified data from `Check token` node.  
    - Outputs:  
      - Donation branch → `Donation` node  
      - Subscription branch → `Subscription` node  
      - Shop Order branch → `Shop Order` node  
    - Edge Cases:  
      - Unknown or missing `type` field (no explicit default branch configured, may cause workflow to stop silently).  
      - Case sensitivity enforced (exact string match required).

#### 2.4 Payment Type Handling

- **Overview:**  
  For each payment type, this block extracts and sets relevant fields from the webhook payload into a structured format for further processing or integration.

- **Nodes Involved:**  
  - `Donation`  
  - `Subscription`  
  - `Shop Order`

- **Node Details:**  
  - **Donation**  
    - Type: Set node  
    - Configuration: Extracts fields: `from_name`, `message`, `amount`, `email` (mapped to `url` field in node), `currency`, `is_public`, and `timestamp` from `$json.body`.  
    - Inputs: From `Check type` node’s Donation branch.  
    - Outputs: Structured donation data.  
    - Edge Cases: Missing fields in payload; no validation beyond mapping.  
    - Sticky Note: "We received a donation. Do your thing."  

  - **Subscription**  
    - Type: Set node  
    - Configuration: Extracts fields: `timestamp`, `from_name`, `message`, `amount`, `url`, `email`, `currency`, `is_first_subscription_payment`, `tier_name`, `is_public`.  
    - Inputs: From `Check type` node’s Subscription branch.  
    - Outputs: Structured subscription data.  
    - Edge Cases: Missing fields; especially `is_first_subscription_payment` which is critical downstream.  
    - Sticky Note: "We received a payment for a subscription. Do your thing."  

  - **Shop Order**  
    - Type: Set node  
    - Configuration: Extracts fields: `from_name`, `amount`, `email`, `currency`, `shop_items` (array), and `url`.  
    - Inputs: From `Check type` node’s Shop Order branch.  
    - Outputs: Structured shop order data.  
    - Edge Cases: Missing or empty `shop_items` array.  
    - Sticky Note: "We received a shop order. Do your thing."  

#### 2.5 Subscription Specific Check

- **Overview:**  
  For subscription payments, this block checks if the payment is the first subscription payment, enabling conditional workflows such as welcome messages or onboarding.

- **Nodes Involved:**  
  - `Is new subscriber?`

- **Node Details:**  
  - **Is new subscriber?**  
    - Type: If node  
    - Configuration: Checks if the field `is_first_subscription_payment` equals string `"true"`.  
    - Inputs: From `Subscription` node.  
    - Outputs:  
      - True branch: indicates a new subscriber (no downstream nodes connected in this workflow, but available for extension).  
      - False branch: no action configured.  
    - Edge Cases:  
      - Field missing or not exactly `"true"` string.  
      - No downstream nodes connected; user must extend for further actions.

---

### 3. Summary Table

| Node Name        | Node Type          | Functional Role                         | Input Node(s)        | Output Node(s)            | Sticky Note                                                                                                  |
|------------------|--------------------|---------------------------------------|----------------------|---------------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook          | Webhook            | Receives incoming Ko-fi webhook POST  | External HTTP POST    | Prepare                   | Setup your webhook. Find your webhook URL in this node and set it [here](https://ko-fi.com/manage/webhooks). |
| Prepare          | Set                | Extracts data and sets verification token | Webhook              | Check token               | Set verification token. Set your Ko-fi verification token in this node. Available [here](https://ko-fi.com/manage/webhooks). |
| Check token      | If                 | Verifies webhook authenticity via token | Prepare              | Check type, Stop and Error |                                                                                                              |
| Stop and Error   | Stop and Error     | Stops workflow on invalid token        | Check token (false)   | None                      |                                                                                                              |
| Check type       | Switch             | Routes workflow based on payment type  | Check token (true)    | Donation, Subscription, Shop Order |                                                                                                              |
| Donation         | Set                | Prepares donation payment data         | Check type (Donation) | None                      | We received a donation. Do your thing.                                                                       |
| Subscription     | Set                | Prepares subscription payment data     | Check type (Subscription) | Is new subscriber?       | We received a payment for a subscription. Do your thing.                                                     |
| Shop Order       | Set                | Prepares shop order payment data       | Check type (Shop Order) | None                    | We received a shop order. Do your thing.                                                                     |
| Is new subscriber? | If                | Checks if subscription payment is first | Subscription          | None                      |                                                                                                              |
| Sticky Note      | Sticky Note        | Instructional note                     | None                 | None                      | Receive and handle Ko-fi payment webhooks with setup instructions and credits.                               |
| Sticky Note1     | Sticky Note        | Instructional note                     | None                 | None                      | Setup your webhook. Find your webhook URL in this node and set it [here](https://ko-fi.com/manage/webhooks). |
| Sticky Note2     | Sticky Note        | Instructional note                     | None                 | None                      | We received a donation. Do your thing.                                                                       |
| Sticky Note3     | Sticky Note        | Instructional note                     | None                 | None                      | We received a payment for a subscription. Do your thing.                                                     |
| Sticky Note4     | Sticky Note        | Instructional note                     | None                 | None                      | We received a shop order. Do your thing.                                                                     |
| Sticky Note5     | Sticky Note        | Instructional note                     | None                 | None                      | Receive and handle Ko-fi payment webhooks with setup instructions and credits.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use a unique identifier (e.g., `83f4e1de-2011-487c-a9f7-be6ccbac0782`)  
   - Purpose: To receive incoming webhook POST requests from Ko-fi.

2. **Create a Set Node named "Prepare"**  
   - Purpose: Extract the `data` object from the webhook payload and set the verification token.  
   - Configuration:  
     - Add a string field `verificationToken` and set its value to your Ko-fi verification token (found in Ko-fi webhook advanced settings).  
     - Add an object field `body` and set its value to `{{$json["body"]["data"]}}` (extracts the main payload).  
   - Connect Webhook node output to this node.

3. **Create an If Node named "Check token"**  
   - Purpose: Verify that the incoming webhook’s `verification_token` matches the stored token.  
   - Condition:  
     - Left value: `{{$json["body"]["verification_token"]}}`  
     - Operator: Equals  
     - Right value: `{{$json["verificationToken"]}}`  
   - Connect "Prepare" node output to this node.

4. **Create a Stop and Error Node named "Stop and Error"**  
   - Purpose: Stop workflow with an error message if token verification fails.  
   - Error message: "Invalid verification token"  
   - Connect the false branch of "Check token" to this node.

5. **Create a Switch Node named "Check type"**  
   - Purpose: Route the workflow based on the payment type.  
   - Property to check: `{{$json["body"]["type"]}}`  
   - Add rules for values:  
     - "Donation"  
     - "Subscription"  
     - "Shop Order"  
   - Connect the true branch of "Check token" to this node.

6. **Create a Set Node named "Donation"**  
   - Purpose: Prepare donation data fields.  
   - Fields to set from `body`:  
     - `from_name` (string) = `{{$json["body"]["from_name"]}}`  
     - `message` (string) = `{{$json["body"]["message"]}}`  
     - `amount` (string) = `{{$json["body"]["amount"]}}`  
     - `url` (string) = `{{$json["body"]["email"]}}` (note: mapped to email)  
     - `currency` (string) = `{{$json["body"]["currency"]}}`  
     - `is_public` (string) = `{{$json["body"]["is_public"]}}`  
     - `timestamp` (string) = `{{$json["body"]["timestamp"]}}`  
   - Connect the "Donation" output of "Check type" to this node.

7. **Create a Set Node named "Subscription"**  
   - Purpose: Prepare subscription data fields.  
   - Fields to set from `body`:  
     - `timestamp` (string) = `{{$json["body"]["timestamp"]}}`  
     - `from_name` (string) = `{{$json["body"]["from_name"]}}`  
     - `message` (string) = `{{$json["body"]["message"]}}`  
     - `amount` (string) = `{{$json["body"]["amount"]}}`  
     - `url` (string) = `{{$json["body"]["url"]}}`  
     - `email` (string) = `{{$json["body"]["email"]}}`  
     - `currency` (string) = `{{$json["body"]["currency"]}}`  
     - `is_first_subscription_payment` (string) = `{{$json["body"]["is_first_subscription_payment"]}}`  
     - `tier_name` (string) = `{{$json["body"]["tier_name"]}}`  
     - `is_public` (string) = `{{$json["body"]["is_public"]}}`  
   - Connect the "Subscription" output of "Check type" to this node.

8. **Create an If Node named "Is new subscriber?"**  
   - Purpose: Check if the subscription payment is the first payment.  
   - Condition:  
     - Left value: `{{$json["is_first_subscription_payment"]}}`  
     - Operator: Equals  
     - Right value: `"true"` (string)  
   - Connect the output of "Subscription" node to this node.  
   - (Optional) Add downstream nodes to handle new subscriber actions.

9. **Create a Set Node named "Shop Order"**  
   - Purpose: Prepare shop order data fields.  
   - Fields to set from `body`:  
     - `from_name` (string) = `{{$json["body"]["from_name"]}}`  
     - `amount` (string) = `{{$json["body"]["amount"]}}`  
     - `email` (string) = `{{$json["body"]["email"]}}`  
     - `currency` (string) = `{{$json["body"]["currency"]}}`  
     - `shop_items` (array) = `{{$json["body"]["shop_items"]}}`  
     - `url` (string) = `{{$json["body"]["url"]}}`  
   - Connect the "Shop Order" output of "Check type" to this node.

10. **Add Sticky Notes for Documentation** (optional but recommended)  
    - Add notes near the `Webhook` and `Prepare` nodes to instruct users on setting webhook URL and verification token with links to Ko-fi settings.  
    - Add notes near each payment type node to indicate where to add custom actions.  
    - Add a general workflow note describing the overall purpose and setup instructions.

11. **Activate the Workflow**  
    - Enable the workflow in n8n to start listening for Ko-fi webhook events.

12. **Test the Workflow**  
    - Use Ko-fi’s webhook test feature to send sample events and verify proper processing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow receives Ko-fi payment webhooks, verifies token, categorizes payment type, and prepares data for further processing.    | General workflow description                                                                    |
| Ko-fi Webhooks Management page for setting webhook URL and verification token.                                                    | https://ko-fi.com/manage/webhooks                                                               |
| Verification token must be set in the `Prepare` node for security.                                                                | Workflow security best practice                                                                 |
| Customize payment type nodes to add actions like sending emails, logging, or notifications.                                        | User customization instructions                                                                 |
| Author: Audun / xqus — Work and shop links: [xqus.com](https://xqus.com), [xqus.gumroad.com](https://xqus.gumroad.com)           | Project credits and author information                                                          |
| Support the author on Ko-fi: [Ko-fi Support Link](https://ko-fi.com/xquscom)                                                      | Support and community engagement                                                                |

---

This documentation provides a thorough understanding of the workflow’s structure, logic, and configuration, enabling advanced users and AI agents to reproduce, modify, and extend it confidently.