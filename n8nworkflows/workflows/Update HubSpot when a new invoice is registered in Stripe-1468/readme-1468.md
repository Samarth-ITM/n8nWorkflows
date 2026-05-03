Update HubSpot when a new invoice is registered in Stripe

https://n8nworkflows.xyz/workflows/update-hubspot-when-a-new-invoice-is-registered-in-stripe-1468


# Update HubSpot when a new invoice is registered in Stripe

### 1. Workflow Overview

This workflow automates the process triggered by a new paid invoice event in Stripe. It performs two primary functions: notifying a Slack channel about the payment and updating the corresponding deal status in HubSpot CRM to reflect the payment. It includes filtering logic to ensure only invoices with a Purchase Order (PO) number are processed further and that the PO number corresponds to an existing HubSpot deal.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception:** Stripe webhook triggers when an invoice payment is successful.
- **1.2 Invoice Validation:** Conditional checks to verify the presence of a PO number and existence of a HubSpot deal with that PO.
- **1.3 HubSpot Deal Update:** Searching for the HubSpot deal by PO number, updating its status to "paid".
- **1.4 Slack Notification:** Posting messages to Slack channels based on payment and validation results.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for Stripe events specifically when an invoice payment succeeds. It initiates the workflow execution.

- **Nodes Involved:**  
  - When Invoice Paid (Stripe Trigger)

- **Node Details:**

  - **When Invoice Paid**  
    - Type: Stripe Trigger (Webhook)  
    - Role: Starts workflow on `invoice.payment_succeeded` event from Stripe.  
    - Configuration: Listens to Stripe API events, specifically the invoice payment succeeded event.  
    - Key expressions: Accesses event data under `$json["data"]["object"]` (invoice details).  
    - Input: External Stripe webhook call.  
    - Output: Passes invoice data downstream.  
    - Edge cases: Failure if Stripe webhook setup is incorrect, webhook URL inaccessible, or Stripe credentials invalid. Additionally, if the event payload format changes, expressions might fail.

---

#### 1.2 Invoice Validation

- **Overview:**  
  This block filters incoming invoices to ensure they contain a PO number and that a corresponding HubSpot deal exists. It routes the workflow accordingly to handle missing data or missing deals.

- **Nodes Involved:**  
  - If no PO Number (IF node)  
  - Send no PO Message (Slack)  
  - Find Deal based on PO Number (HubSpot Search)  
  - If no deal found for PO (IF node)  
  - Send Deal not found message (Slack)

- **Node Details:**

  - **If no PO Number**  
    - Type: IF node  
    - Role: Checks if the invoice custom fields array (where PO number is expected) is empty.  
    - Configuration: Condition `isEmpty` on `$json["data"]["object"]["custom_fields"]`.  
    - Input: Output from Stripe Trigger.  
    - Output:  
      - True branch: No PO number → triggers Slack notification "Send no PO Message".  
      - False branch: PO number present → proceeds to "Find Deal based on PO Number".  
    - Edge cases: Expression might fail if `custom_fields` is missing or unexpectedly formatted.

  - **Send no PO Message**  
    - Type: Slack node  
    - Role: Posts a message alerting the team that a Stripe payment was received without a PO number.  
    - Configuration: Posts in channel `team-accounts`. Message includes invoice amount, currency, customer name/email, and transaction ID.  
    - Input: True branch from "If no PO Number".  
    - Edge cases: Slack API auth failure, invalid channel, or rate limits.

  - **Find Deal based on PO Number**  
    - Type: HubSpot node (Search)  
    - Role: Searches HubSpot deals filtering by `po_number` property matching the invoice's PO number.  
    - Configuration: Filter by `po_number` equal to `$json["data"]["object"]["custom_fields"][0]["value"]`.  
    - Credentials: Uses HubSpot API with API key or OAuth2 (depending on config).  
    - Input: False branch from "If no PO Number".  
    - Output: Passes deal data downstream.  
    - Edge cases: Search failure due to invalid HubSpot credentials, rate limits, or if PO number property is missing or misconfigured.

  - **If no deal found for PO**  
    - Type: IF node  
    - Role: Checks if the search result returned no deals by testing if `$json["id"]` is empty.  
    - Input: Output from "Find Deal based on PO Number".  
    - Output:  
      - True branch: No deal → triggers Slack notification "Send Deal not found message".  
      - False branch: Deal found → continues to update deal.  
    - Edge cases: Expression failure if response structure is unexpected.

  - **Send Deal not found message**  
    - Type: Slack node  
    - Role: Notifies the team that no HubSpot deal was found matching the PO number for the invoice.  
    - Configuration: Posts in channel `team-accounts` with payment details and PO number.  
    - Input: True branch from "If no deal found for PO".  
    - Edge cases: Slack API errors, invalid channel.

---

#### 1.3 HubSpot Deal Update

- **Overview:**  
  If a HubSpot deal matching the PO number is found, this block updates the deal's "paid" status to "Yes" to reflect the invoice payment.

- **Nodes Involved:**  
  - Update Deal to Paid (HubSpot Update)

- **Node Details:**

  - **Update Deal to Paid**  
    - Type: HubSpot node (Update)  
    - Role: Updates the specified deal's custom property `paid` to "Yes".  
    - Configuration:  
      - Uses dynamic expression for deal ID: `{{$json["id"]}}` from previous search node.  
      - Sets `paid` property to "Yes".  
      - Authenticates via OAuth2.  
    - Input: False branch from "If no deal found for PO".  
    - Output: Passes updated deal data downstream.  
    - Edge cases: Failure to update due to invalid deal ID, authentication issues, or API rate limits.

---

#### 1.4 Slack Notification

- **Overview:**  
  This block sends confirmation messages to Slack about the invoice payment being processed and HubSpot deal updated.

- **Nodes Involved:**  
  - Send invoice paid message (Slack)

- **Node Details:**

  - **Send invoice paid message**  
    - Type: Slack node  
    - Role: Posts a celebratory message to the `team-accounts` channel indicating an invoice was paid and the associated deal updated.  
    - Configuration:  
      - Message contains invoice amount (converted from cents to units), currency, customer name/email, PO number, and transaction ID.  
      - Visual attachments with green color (`#00FF04`) for success.  
    - Input: Output from "Update Deal to Paid".  
    - Edge cases: Slack API failures, invalid channel or message formatting errors.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                  | Input Node(s)           | Output Node(s)                  | Sticky Note                      |
|-------------------------|---------------------|-------------------------------------------------|------------------------|-------------------------------|---------------------------------|
| When Invoice Paid       | Stripe Trigger      | Trigger on new successful invoice payment       | -                      | If no PO Number                |                                 |
| If no PO Number         | IF                  | Checks if invoice has PO number                   | When Invoice Paid       | Send no PO Message, Find Deal based on PO Number |                                 |
| Send no PO Message      | Slack               | Notify that payment has no PO number              | If no PO Number (true)  | -                             |                                 |
| Find Deal based on PO Number | HubSpot (Search) | Searches for deal by PO number                     | If no PO Number (false) | If no deal found for PO        |                                 |
| If no deal found for PO | IF                  | Checks if deal found for PO number                 | Find Deal based on PO Number | Send Deal not found message, Update Deal to Paid |                                 |
| Send Deal not found message | Slack            | Notify that no HubSpot deal was found              | If no deal found for PO (true) | -                             |                                 |
| Update Deal to Paid     | HubSpot (Update)    | Update deal status to "paid"                        | If no deal found for PO (false) | Send invoice paid message   |                                 |
| Send invoice paid message | Slack             | Notify team that invoice was paid and deal updated | Update Deal to Paid     | -                             |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named appropriately (e.g., "On new Stripe Invoice Payment update Hubspot and notify the team in Slack").

2. **Add Stripe Trigger node**:  
   - Set node name to "When Invoice Paid".  
   - Select event: `invoice.payment_succeeded`.  
   - Configure with valid Stripe credentials (API key).  
   - This node listens for Stripe webhook events on invoice payment success.

3. **Add IF node** named "If no PO Number":  
   - Condition: Check if the invoice's `custom_fields` is empty.  
   - Expression: `{{$json["data"]["object"]["custom_fields"]}}` with operation `isEmpty`.  
   - Connect "When Invoice Paid" → "If no PO Number".

4. **Add Slack node** named "Send no PO Message":  
   - Set channel to `team-accounts`.  
   - Compose message with attachments including amount (`{{$json["data"]["object"]["amount_paid"] / 100}}`), currency, customer name/email, and transaction ID.  
   - Configure with Slack credentials (OAuth or token).  
   - Connect "If no PO Number" True branch → "Send no PO Message".

5. **Add HubSpot node (Search) named "Find Deal based on PO Number"**:  
   - Operation: Search deals.  
   - Filter: `po_number` equal to `{{$json["data"]["object"]["custom_fields"][0]["value"]}}`.  
   - Use HubSpot API credentials.  
   - Connect "If no PO Number" False branch → "Find Deal based on PO Number".

6. **Add IF node named "If no deal found for PO"**:  
   - Condition: Check if `id` is empty (`{{$json["id"]}}` is empty).  
   - Connect "Find Deal based on PO Number" → "If no deal found for PO".

7. **Add Slack node named "Send Deal not found message"**:  
   - Channel: `team-accounts`.  
   - Message: Include amount, currency, customer info, PO number, and transaction ID similar to previous Slack node but indicating deal not found.  
   - Connect "If no deal found for PO" True branch → "Send Deal not found message".

8. **Add HubSpot node (Update) named "Update Deal to Paid"**:  
   - Operation: Update deal.  
   - Deal ID: `{{$json["id"]}}` from the HubSpot search node.  
   - Update field `paid` to "Yes".  
   - Authenticate with HubSpot OAuth2 credentials.  
   - Connect "If no deal found for PO" False branch → "Update Deal to Paid".

9. **Add Slack node named "Send invoice paid message"**:  
   - Channel: `team-accounts`.  
   - Message with green success color and fields: amount, currency, customer name/email, PO number, and transaction ID.  
   - Connect "Update Deal to Paid" → "Send invoice paid message".

10. **Activate the workflow** and ensure the Stripe webhook URL is registered correctly in your Stripe dashboard.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Prerequisites include valid Slack, HubSpot, and Stripe accounts with proper OAuth or API credential setup.    | See n8n integrations documentation: Slack, HubSpot, Stripe credentials setup.                        |
| Slack channel used throughout is `team-accounts`, replace as needed for your team communication preferences.  | Slack API channel naming and permissions.                                                           |
| HubSpot property used for filtering is `po_number`, must be configured in HubSpot deals with correct data type. | HubSpot CRM customization for properties: https://developers.hubspot.com/docs/api/crm/properties     |
| The workflow handles cases with missing PO number or missing HubSpot deals by notifying the team in Slack.    | Useful for operational visibility and error handling.                                               |
| Amount values are converted from cents to units by dividing by 100.                                           | Stripe invoice amounts are in cents by default.                                                     |

---

This documentation fully describes the workflow's logic, node configurations, dependencies, and error handling paths, allowing for accurate reproduction, modification, and troubleshooting.