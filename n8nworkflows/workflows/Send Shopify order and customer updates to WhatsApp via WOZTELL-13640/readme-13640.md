Send Shopify order and customer updates to WhatsApp via WOZTELL

https://n8nworkflows.xyz/workflows/send-shopify-order-and-customer-updates-to-whatsapp-via-woztell-13640


# Send Shopify order and customer updates to WhatsApp via WOZTELL

# 1. Workflow Overview

This workflow listens to selected Shopify events and sends a WhatsApp template message to the customer through WOZTELL when a phone number is available.

Its main purpose is to automate customer-facing notifications from Shopify, especially for ecommerce scenarios such as:

- notifying customers after an order is paid
- sending a welcome or onboarding message after customer creation
- sending fulfillment-related updates

The workflow is intentionally simple: three Shopify webhook triggers feed a shared validation step, and successful items are passed to a WOZTELL node that sends a pre-approved WhatsApp template.

## 1.1 Event Reception from Shopify

This block contains three independent Shopify Trigger nodes. Each one listens to a different Shopify event and starts the workflow when that event occurs.

## 1.2 Phone Validation

This block checks whether the incoming payload includes a top-level `phone` field. Only items that pass this check continue to message sending.

## 1.3 WhatsApp Template Delivery via WOZTELL

This block sends a WhatsApp template message using the WOZTELL node. It maps incoming Shopify data to template variables and uses the detected phone number as the recipient identifier.

---

# 2. Block-by-Block Analysis

## 2.1 Event Reception from Shopify

### Overview

This block provides three entry points into the workflow. Each trigger is a webhook-based Shopify listener configured for a specific event topic, allowing the workflow to react to different customer or order lifecycle updates.

### Nodes Involved

- Shopify Trigger - Order Paid
- Shopify Trigger - Customer Created
- Shopify Trigger - Fulfillment Created

### Node Details

#### Shopify Trigger - Order Paid

- **Type and technical role:** `n8n-nodes-base.shopifyTrigger`  
  Webhook trigger node for Shopify events. It starts the workflow when Shopify emits the `orders/paid` event.

- **Configuration choices:**
  - Topic: `orders/paid`
  - Authentication: OAuth2
  - Uses a generated webhook ID managed by n8n

- **Key expressions or variables used:**  
  None in node configuration.

- **Input and output connections:**
  - Input: none, this is an entry point
  - Output: connected to `If`

- **Version-specific requirements:**
  - Node type version: `1`
  - Requires a compatible Shopify Trigger node in the installed n8n version

- **Edge cases or potential failure types:**
  - Shopify OAuth2 credential misconfiguration
  - Webhook registration failure
  - Missing required Shopify app scopes
  - Payload shape mismatch for downstream nodes
  - In the sample pinned data, the top-level `phone` exists, but `customer.phone` is `null`; downstream variable mapping uses `customer.first_name` and `order_number`, which is valid for this trigger

- **Sub-workflow reference:**  
  None

---

#### Shopify Trigger - Customer Created

- **Type and technical role:** `n8n-nodes-base.shopifyTrigger`  
  Webhook trigger for newly created Shopify customers.

- **Configuration choices:**
  - Topic: `customers/create`
  - Authentication: OAuth2

- **Key expressions or variables used:**  
  None in node configuration.

- **Input and output connections:**
  - Input: none, this is an entry point
  - Output: connected to `If`

- **Version-specific requirements:**
  - Node type version: `1`

- **Edge cases or potential failure types:**
  - OAuth2 or webhook subscription issues
  - Missing customer phone at top level causes the workflow to stop at the `If` node
  - This trigger’s payload usually contains `first_name`, but not `customer.first_name`
  - The WOZTELL node expects `{{ $json.customer.first_name }}` and `{{ $json.order_number }}`, which do **not** exist for a typical customer creation payload; if the `If` condition passes, the message node may send empty or invalid template variables unless reconfigured

- **Sub-workflow reference:**  
  None

---

#### Shopify Trigger - Fulfillment Created

- **Type and technical role:** `n8n-nodes-base.shopifyTrigger`  
  Webhook trigger for Shopify fulfillment event creation.

- **Configuration choices:**
  - Topic: `fulfillment_events/create`
  - Authentication: OAuth2

- **Key expressions or variables used:**  
  None in node configuration.

- **Input and output connections:**
  - Input: none, this is an entry point
  - Output: connected to `If`

- **Version-specific requirements:**
  - Node type version: `1`

- **Edge cases or potential failure types:**
  - OAuth2 or webhook setup issues
  - The sample fulfillment payload shown in pin data has no top-level `phone`, so the `If` node will usually block execution
  - This payload also does not contain `customer.first_name` or `order_number` directly, so the WOZTELL node is not structurally aligned with this trigger unless enrichment steps are added first
  - Depending on the Shopify event, the payload may require an additional Shopify lookup node to retrieve order/customer details

- **Sub-workflow reference:**  
  None

---

## 2.2 Phone Validation

### Overview

This block acts as a safety gate before WhatsApp delivery. It ensures the payload contains a top-level `phone` property so the workflow does not attempt to send a message without a recipient identifier.

### Nodes Involved

- If

### Node Details

#### If

- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional routing node used to validate the existence of a phone number.

- **Configuration choices:**
  - Uses condition-combination mode `and`
  - Evaluates one condition:
    - left value: `{{ $json.phone }}`
    - operator: string `exists`
  - Type validation is loose

- **Key expressions or variables used:**
  - `{{ $json.phone }}`

- **Input and output connections:**
  - Inputs:
    - `Shopify Trigger - Order Paid`
    - `Shopify Trigger - Customer Created`
    - `Shopify Trigger - Fulfillment Created`
  - Output:
    - True branch connected to `Send message template`
    - False branch unused

- **Version-specific requirements:**
  - Node type version: `2.2`
  - Condition format uses the newer IF node structure with condition options version `2`

- **Edge cases or potential failure types:**
  - It only checks top-level `phone`, not nested values such as `customer.phone` or `default_address.phone`
  - For Shopify order payloads, phone may be present in multiple places and the chosen field may not be the best WhatsApp-compatible number
  - A value may "exist" but still be invalid for WhatsApp formatting
  - Fulfillment payloads usually will not contain a direct phone number, so this branch will commonly stop there
  - Because the false branch is not connected, failed items are silently dropped

- **Sub-workflow reference:**  
  None

---

## 2.3 WhatsApp Template Delivery via WOZTELL

### Overview

This block sends a pre-approved WhatsApp message template to the phone number that passed validation. It relies on WOZTELL credentials and maps Shopify fields into the template variables required by the chosen WhatsApp template.

### Nodes Involved

- Send message template

### Node Details

#### Send message template

- **Type and technical role:** `@woztell-sanuker/n8n-nodes-woztell-sanuker.woztell`  
  Third-party WOZTELL integration node used here for WhatsApp template sending.

- **Configuration choices:**
  - Operation: `sendTemplates`
  - Recipient ID: `{{ $json.phone }}`
  - Variables explicitly mapped:
    - Variable #1: `{{ $json.customer.first_name }}`
    - Variable #2: `{{ $json.order_number }}`
  - Channel is configured through a selectable resource list, but no concrete channel value is present in the exported JSON
  - Headers, buttons, and carousel sections are present but empty
  - Uses WOZTELL API credentials

- **Key expressions or variables used:**
  - `{{ $json.phone }}`
  - `{{ $json.customer.first_name }}`
  - `{{ $json.order_number }}`

- **Input and output connections:**
  - Input: true branch from `If`
  - Output: none

- **Version-specific requirements:**
  - Node type version: `1`
  - Requires the custom package `@woztell-sanuker/n8n-nodes-woztell-sanuker` to be installed in the n8n instance
  - Requires valid WOZTELL credentials of type `woztellCredentialApi`

- **Edge cases or potential failure types:**
  - Missing or invalid WOZTELL credentials
  - Missing channel selection
  - Selected WhatsApp template may require different variables than the two mapped here
  - `customer.first_name` and `order_number` do not exist for all incoming trigger payloads
  - `phone` may not be in E.164 or channel-compatible format
  - Template approval or channel permission problems in WOZTELL
  - API permission gaps, especially if the required token scopes are missing

- **Sub-workflow reference:**  
  None

---

## 2.4 Documentation and On-Canvas Guidance

### Overview

This workflow includes several Sticky Note nodes that document setup expectations, usage guidance, credential requirements, and related external resources. These notes do not execute but are important for correct deployment.

### Nodes Involved

- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note

### Node Details

#### Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation-only node describing Shopify trigger setup and required scopes.

- **Configuration choices:**
  - Positioned around the trigger area
  - Explains which Shopify webhook events to enable
  - Lists Shopify API scopes:
    - `read_customer_events`
    - `read_customers`
    - `read_orders`
    - `read_fulfillments`

- **Input and output connections:** none

- **Version-specific requirements:** type version `1`

- **Edge cases or potential failure types:**
  - If scopes are not granted, trigger registration or data access may fail

- **Sub-workflow reference:** None

---

#### Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation note for the IF validation step.

- **Configuration choices:**
  - Explains that processing continues only when a phone number exists

- **Input and output connections:** none

- **Version-specific requirements:** type version `1`

- **Edge cases or potential failure types:**
  - Note is conceptually correct, but actual implementation only checks top-level `phone`

- **Sub-workflow reference:** None

---

#### Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation note for WOZTELL message sending and credential/token setup.

- **Configuration choices:**
  - Explains template selection and variable mapping
  - Includes support links for access token generation
  - Lists required permissions:
    - `channel:list`
    - `channel:getBasicInfo`
    - `channel:getEnvironmentInfo`
    - `channel:getDetails`
    - `bot:sendResponses`

- **Input and output connections:** none

- **Version-specific requirements:** type version `1`

- **Edge cases or potential failure types:**
  - Missing permissions will prevent listing channels or sending messages

- **Sub-workflow reference:** None

---

#### Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Large informational note describing workflow purpose, requirements, customization ideas, and WOZTELL onboarding resources.

- **Configuration choices:**
  - Includes onboarding and signup links
  - Provides business context and usage guidance

- **Input and output connections:** none

- **Version-specific requirements:** type version `1`

- **Edge cases or potential failure types:** none directly; informational only

- **Sub-workflow reference:** None

---

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Supplementary documentation note with text and video setup resources.

- **Configuration choices:**
  - Includes text guide link
  - Includes YouTube video reference

- **Input and output connections:** none

- **Version-specific requirements:** type version `1`

- **Edge cases or potential failure types:** informational only

- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify Trigger - Order Paid | n8n-nodes-base.shopifyTrigger | Entry point for paid order events from Shopify |  | If | ### 1) Listens for a selected Shopify event.  \nYou can enable or disable specific triggers depending on what notifications you want to send.  \n\nFor this template, create webhooks for:  \n- Order creation  \n- Customer creation  \n- Fulfillment creation  \n##  \nMake sure your Shopify app includes these API scopes:  \n- read_customer_events  \n- read_customers  \n- read_orders  \n- read_fulfillments |
| Shopify Trigger - Customer Created | n8n-nodes-base.shopifyTrigger | Entry point for customer creation events from Shopify |  | If | ### 1) Listens for a selected Shopify event.  \nYou can enable or disable specific triggers depending on what notifications you want to send.  \n\nFor this template, create webhooks for:  \n- Order creation  \n- Customer creation  \n- Fulfillment creation  \n##  \nMake sure your Shopify app includes these API scopes:  \n- read_customer_events  \n- read_customers  \n- read_orders  \n- read_fulfillments |
| Shopify Trigger - Fulfillment Created | n8n-nodes-base.shopifyTrigger | Entry point for fulfillment event creation from Shopify |  | If | ### 1) Listens for a selected Shopify event.  \nYou can enable or disable specific triggers depending on what notifications you want to send.  \n\nFor this template, create webhooks for:  \n- Order creation  \n- Customer creation  \n- Fulfillment creation  \n##  \nMake sure your Shopify app includes these API scopes:  \n- read_customer_events  \n- read_customers  \n- read_orders  \n- read_fulfillments |
| If | n8n-nodes-base.if | Validates that a phone number exists before sending | Shopify Trigger - Order Paid, Shopify Trigger - Customer Created, Shopify Trigger - Fulfillment Created | Send message template | ### 2) Checks whether the customer has a phone number.\n\nIf a phone number exists, the workflow continues.\n\nIf not, it stops to prevent sending errors. |
| Send message template | @woztell-sanuker/n8n-nodes-woztell-sanuker.woztell | Sends WhatsApp template messages via WOZTELL | If |  | ### 3) Sends a WhatsApp template message to the customer.\n \nSelect your sending channel, select an approved template and map Shopify data (like customer name and order number) into template variables.\n\nSet up your WOZTELL credentials. To generate the access token, follow the step-by-step guide [here](https://support.woztell.com/portal/en/kb/articles/access-token#Access_Token_Generation).\n\nRequired permissions:\n- channel:list\n- channel:getBasicInfo\n- channel:getEnvironmentInfo\n- channel:getDetails\nbot:sendResponses\n##\nFor the WOZTELL Channel API field, follow [this guide](https://support.woztell.com/portal/en/kb/articles/access-token-channels-in-woztell#How_to_Create_an_Access_Token_for_a_Channel_in_WOZTELL) to create a new token. If you have already generated a token, you may reuse it. |
| Sticky Note1 | n8n-nodes-base.stickyNote | On-canvas documentation for Shopify trigger setup |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | On-canvas documentation for phone validation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | On-canvas documentation for WOZTELL configuration |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | General overview, requirements, and external resources |  |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | Additional setup resources and video link |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the first Shopify trigger node.**
   - Node type: `Shopify Trigger`
   - Name it: `Shopify Trigger - Order Paid`
   - Set:
     - **Topic**: `orders/paid`
     - **Authentication**: `OAuth2`
   - Create or select Shopify OAuth2 credentials.
   - Ensure the Shopify app has these scopes:
     - `read_customer_events`
     - `read_customers`
     - `read_orders`
     - `read_fulfillments`

3. **Add the second Shopify trigger node.**
   - Node type: `Shopify Trigger`
   - Name it: `Shopify Trigger - Customer Created`
   - Set:
     - **Topic**: `customers/create`
     - **Authentication**: `OAuth2`
   - Reuse the same Shopify OAuth2 credentials if appropriate.

4. **Add the third Shopify trigger node.**
   - Node type: `Shopify Trigger`
   - Name it: `Shopify Trigger - Fulfillment Created`
   - Set:
     - **Topic**: `fulfillment_events/create`
     - **Authentication**: `OAuth2`

5. **Add an IF node for phone validation.**
   - Node type: `If`
   - Name it: `If`
   - Configure one condition:
     - Left value: `{{ $json.phone }}`
     - Operator: `exists`
   - Leave combinator as `and`

6. **Connect all three Shopify trigger nodes into the IF node.**
   - `Shopify Trigger - Order Paid` → `If`
   - `Shopify Trigger - Customer Created` → `If`
   - `Shopify Trigger - Fulfillment Created` → `If`

7. **Install the WOZTELL custom node package if it is not already available in your n8n instance.**
   - Required node package:
     - `@woztell-sanuker/n8n-nodes-woztell-sanuker`
   - This step depends on your n8n hosting model. Self-hosted instances may require manual community-node installation.

8. **Add the WOZTELL node.**
   - Node type: `WOZTELL`
   - Name it: `Send message template`
   - Set the operation to:
     - `sendTemplates`

9. **Configure WOZTELL credentials.**
   - Create credential type: `WOZTELL account` / `woztellCredentialApi`
   - Use an access token generated from WOZTELL
   - Required permissions mentioned in the workflow notes:
     - `channel:list`
     - `channel:getBasicInfo`
     - `channel:getEnvironmentInfo`
     - `channel:getDetails`
     - `bot:sendResponses`
   - Also configure the WOZTELL Channel API token if required by your account setup

10. **Configure the WOZTELL message node fields.**
    - **Recipient ID**: `{{ $json.phone }}`
    - **Channel**: choose your WhatsApp sending channel from WOZTELL
    - **Template**: choose an approved WhatsApp template in WOZTELL
    - **Variables**:
      - Variable #1: `{{ $json.customer.first_name }}`
      - Variable #2: `{{ $json.order_number }}`

11. **Leave optional WOZTELL sections empty unless your template requires them.**
    - Headers: empty
    - Buttons: empty
    - Carousel: empty

12. **Connect the IF node true output to the WOZTELL node.**
    - `If` true branch → `Send message template`

13. **Do not connect the false branch unless you want explicit error handling or logging.**
    - In the provided workflow, items without a phone number simply stop.

14. **Test each trigger individually.**
    - Use Shopify test events or real events
    - Confirm whether the payload contains:
      - top-level `phone`
      - `customer.first_name`
      - `order_number`

15. **Review payload compatibility before activating.**
    - This is important because the current shared WOZTELL mapping is best suited to the `orders/paid` payload.
    - For `customers/create`, the payload usually contains:
      - `first_name`
      - possibly `phone`
      - but not `customer.first_name` or `order_number`
    - For `fulfillment_events/create`, the payload often lacks:
      - `phone`
      - `customer.first_name`
      - `order_number`
    - To make all triggers fully functional, add normalization or lookup nodes, such as:
      - a `Set` node to unify fields
      - a Shopify lookup node to fetch the related order/customer
      - separate IF and WOZTELL branches per event type

16. **Activate the workflow.**

17. **Update Shopify webhook usage if needed.**
    - As noted in the canvas instructions, switch the webhook from test mode or test URL to the production URL once validated.

## Credential Setup Expectations

### Shopify OAuth2

- Must be authorized against the target Shopify store
- Must support webhook trigger registration
- Must include required scopes:
  - `read_customer_events`
  - `read_customers`
  - `read_orders`
  - `read_fulfillments`

### WOZTELL

- Requires an active WOZTELL account
- Requires WhatsApp Business Platform activation
- Requires at least one approved message template
- Requires API token permissions for channel listing and bot response sending

## No Sub-Workflow Usage

This workflow does not invoke any sub-workflows and does not use the Execute Workflow node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically sends WhatsApp notifications from Shopify using WOZTELL. It listens for Shopify events such as order paid, customer created, or fulfillment created, then sends a pre approved WhatsApp template to the customer. | General workflow purpose |
| Designed for ecommerce businesses that want to automatically send WhatsApp order and fulfillment updates to customers. Suitable for store owners or agencies managing Shopify automation. | Target users |
| Import into n8n, connect Shopify with OAuth, connect WOZTELL with an access token, select a WhatsApp template, map Shopify fields, activate the workflow, then switch Shopify webhooks from Test URL to Production URL. | Operational guidance |
| Requirements: Shopify store admin access, n8n account, WOZTELL paid account with WhatsApp Business Platform activated, approved WhatsApp message template in WOZTELL. | Deployment requirements |
| Customization ideas: change Shopify trigger events, add more IF conditions, use different WhatsApp templates for each event, add Slack/email/CRM steps, or replace Shopify with another ecommerce trigger source. | Extension ideas |
| WOZTELL signup link | https://platform.woztell.com/signup?lang=en&utm_campaign=plugin-n8n&utm_medium=plugin-n8n&utm_source=N8N |
| WhatsApp Business API setup guide | https://doc.woztell.com/docs/procedures/basic-whatsapp-chatbot-setup/standard-procedures-wa-connect-waba/ |
| Text setup guide for this workflow pattern | https://woztell.com/connect-shopify-whatsapp-n8n/ |
| WOZTELL access token generation guide | https://support.woztell.com/portal/en/kb/articles/access-token#Access_Token_Generation |
| WOZTELL channel token guide | https://support.woztell.com/portal/en/kb/articles/access-token-channels-in-woztell#How_to_Create_an_Access_Token_for_a_Channel_in_WOZTELL |
| Video guide | YouTube: `Av5mVz21D28` |

## Important Implementation Note

Although the workflow exposes three Shopify entry points, the message mapping is not normalized across all three payload types. In its current form:

- **Order Paid** is the only trigger clearly aligned with the message variables
- **Customer Created** may pass the phone check but still fail or send incomplete template variables
- **Fulfillment Created** will usually stop at the IF node because no top-level phone is present

For production use across all three events, the safest redesign is to create one branch per event type or add a normalization layer before the WOZTELL node.