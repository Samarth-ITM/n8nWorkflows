Recover abandoned WooCommerce carts using OpenAI GPT-4.1-mini, Gmail and Slack

https://n8nworkflows.xyz/workflows/recover-abandoned-woocommerce-carts-using-openai-gpt-4-1-mini--gmail-and-slack-14367


# Recover abandoned WooCommerce carts using OpenAI GPT-4.1-mini, Gmail and Slack

# 1. Workflow Overview

This workflow is designed to recover potentially abandoned WooCommerce carts. It receives cart data from a store via webhook, waits for a delay period, rechecks the cart through an API, and if the cart is still unpaid and still contains items, it uses OpenAI GPT-4.1-mini to generate a personalized recovery email. The workflow then sends that email through Gmail and notifies the internal team in Slack.

Typical use cases:
- Recovering sales from shoppers who left without completing checkout
- Automatically generating personalized follow-up emails
- Alerting a sales or ecommerce team when recovery actions are triggered

## 1.1 Input Reception and Cart Normalization
The workflow begins when the store sends cart information to an n8n webhook. The incoming payload is normalized into a simpler internal structure.

## 1.2 Delay and Cart Revalidation
After receiving the event, the workflow waits for a configured period, then queries the store again using the cart key to confirm whether the cart is still abandoned.

## 1.3 Abandonment Decision
An IF node checks whether the cart still contains items and whether payment is still required. Only carts that satisfy both conditions continue.

## 1.4 AI Email Generation
If the cart is still abandoned, an AI agent uses the updated cart details to generate a short, personalized reminder email.

## 1.5 Email Formatting and Outbound Actions
The AI output is split into subject and body. The workflow then sends the email to the customer through Gmail and sends a Slack notification to the internal team.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Cart Normalization

### Overview
This block receives abandoned-cart event data from the store and converts raw webhook fields into a cleaner structure for later nodes. It is the entry point for the workflow.

### Nodes Involved
- Receive Cart Event1
- Normalize Incoming Cart Data1
- Sticky Note6
- Sticky Note7
- Sticky Note11

### Node Details

#### Receive Cart Event1
- **Type and technical role:** `n8n-nodes-base.webhook` — entry-point webhook that receives HTTP requests from the ecommerce store.
- **Configuration choices:**
  - Webhook path: `abandoned-cart`
  - No special webhook options are configured
- **Key expressions or variables used:** None inside the node itself
- **Input and output connections:**
  - Input: none
  - Output: `Normalize Incoming Cart Data1`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Store sends data to the wrong webhook path
  - Payload shape does not match expected `body.*` fields
  - Webhook is inactive or test/production URL confusion
- **Sub-workflow reference:** None

#### Normalize Incoming Cart Data1
- **Type and technical role:** `n8n-nodes-base.set` — maps incoming webhook payload fields into normalized fields used later.
- **Configuration choices:**
  - Creates these fields:
    - `cart_key` from `{{$json.body.cart_key}}`
    - `email` from `{{$json.body.email}}`
    - `first_name` from `{{$json.body.first_name}}`
    - `cart_total` from `{{$json.body.cart_total}}`
    - `item_name` from `{{$json.body.item_name}}`  
      - Note: the configured field name appears as `=item_name`, which is likely accidental and should be corrected to `item_name`
    - `item_quantity` from `{{$json.body.item_quantity}}`
- **Key expressions or variables used:**
  - `{{$json.body.cart_key}}`
  - `{{$json.body.email}}`
  - `{{$json.body.first_name}}`
  - `{{$json.body.cart_total}}`
  - `{{$json.body.item_name}}`
  - `{{$json.body.item_quantity}}`
- **Input and output connections:**
  - Input: `Receive Cart Event1`
  - Output: `Wait before checking cart status1`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - Missing `body` object in webhook payload
  - Missing individual fields causing empty strings or undefined values
  - Accidental field-name typo (`=item_name`) may produce unexpected output schema
- **Sub-workflow reference:** None

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation
- **Configuration choices:** Explains that the website sends cart details to the webhook when a user leaves without checkout
- **Key expressions or variables used:** None
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation
- **Configuration choices:** Notes that the Set node cleans and organizes incoming cart data
- **Key expressions or variables used:** None
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note11
- **Type and technical role:** `n8n-nodes-base.stickyNote` — high-level workflow documentation
- **Configuration choices:** Contains a full explanation of workflow logic and setup steps
- **Key expressions or variables used:** None
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

---

## 2.2 Delay and Cart Revalidation

### Overview
This block gives the customer time to finish checkout, then checks the store again using the cart key. It is the verification stage before any reminder is sent.

### Nodes Involved
- Wait before checking cart status1
- Recheck cart status from store1
- Sticky Note8
- Sticky Note11

### Node Details

#### Wait before checking cart status1
- **Type and technical role:** `n8n-nodes-base.wait` — pauses workflow execution before rechecking the cart.
- **Configuration choices:**
  - No explicit wait duration is visible in the exported JSON
  - This means the node must be configured manually after import or may currently rely on default/incomplete settings
- **Key expressions or variables used:** None shown
- **Input and output connections:**
  - Input: `Normalize Incoming Cart Data1`
  - Output: `Recheck cart status from store1`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**
  - If no delay is configured, behavior may not match intended cart-recovery timing
  - Wait executions require n8n persistence/execution-resume support
  - Workflow resumption may fail if instance configuration does not support waiting nodes correctly
- **Sub-workflow reference:** None

#### Recheck cart status from store1
- **Type and technical role:** `n8n-nodes-base.httpRequest` — queries the store’s cart API to get current cart state.
- **Configuration choices:**
  - URL is a placeholder: `{{your_cart_API_here}}`
  - Sends a request header:
    - Header name: ` cocart-cart-key`
    - Header value: `={{$json.cart_key}}`
  - No advanced request options are configured
- **Key expressions or variables used:**
  - `{{$json.cart_key}}`
- **Input and output connections:**
  - Input: `Wait before checking cart status1`
  - Output: `Cart is Still Abandoned?1`
- **Version-specific requirements:** Type version `4.3`
- **Edge cases or potential failure types:**
  - Placeholder URL must be replaced with a real CoCart/store API endpoint
  - Header name contains a leading space (` cocart-cart-key`), which may break the API request and should be corrected to `cocart-cart-key`
  - Authentication may be required depending on store setup
  - API timeout, DNS failure, 4xx/5xx responses
  - API response structure must contain fields later used by downstream nodes such as `item_count`, `needs_payment`, `items`, and `customer.billing_address.*`
- **Sub-workflow reference:** None

#### Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation
- **Configuration choices:** Describes wait, verification, AI generation, formatting, email send, and Slack notification logic
- **Key expressions or variables used:** None
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note11
- Same general role as above; it also documents this block at a broader level.

---

## 2.3 Abandonment Decision

### Overview
This block determines whether the workflow should proceed. It only continues when the cart still has items and payment is still outstanding.

### Nodes Involved
- Cart is Still Abandoned?1
- Sticky Note8
- Sticky Note11

### Node Details

#### Cart is Still Abandoned?1
- **Type and technical role:** `n8n-nodes-base.if` — conditional gate for abandonment validation
- **Configuration choices:**
  - Uses AND logic with two conditions:
    1. `{{$json.item_count}} > 0`
    2. `{{$json.needs_payment}}` is `true`
- **Key expressions or variables used:**
  - `{{$json.item_count}}`
  - `{{$json.needs_payment}}`
- **Input and output connections:**
  - Input: `Recheck cart status from store1`
  - True output: `Genrate Personalized Reminder Email1`
  - False output: no connection, so the workflow stops silently
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**
  - Response values may have unexpected types, especially if `item_count` is a string instead of a number
  - `needs_payment` may be absent or not strictly boolean
  - If API schema differs from expected WooCommerce/CoCart response shape, the condition may fail incorrectly
- **Sub-workflow reference:** None

---

## 2.4 AI Email Generation

### Overview
This block generates a personalized recovery email using customer and cart information returned by the recheck API. It relies on an OpenAI chat model attached to an AI Agent node.

### Nodes Involved
- OpenAI Chat Model1
- Genrate Personalized Reminder Email1
- Sticky Note8
- Sticky Note11

### Node Details

#### OpenAI Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI chat model provider used by the agent node
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - No additional options are configured
  - Uses OpenAI credentials named `OpenAi account 17`
- **Key expressions or variables used:** None in this node
- **Input and output connections:**
  - Connected to `Genrate Personalized Reminder Email1` through the `ai_languageModel` port
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - Invalid or expired OpenAI API credentials
  - Model availability or quota limits
  - Regional/model access restrictions
  - Latency or rate limiting
- **Sub-workflow reference:** None

#### Genrate Personalized Reminder Email1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — AI agent that prompts the model to generate reminder email text
- **Configuration choices:**
  - Prompt type: defined directly in node
  - Main prompt:
    - Uses customer first name from `customer.billing_address.billing_first_name`
    - Uses first cart item name from `items[0].name`
    - Uses first item total from `items[0].totals.total`
    - Requests output under 120 words
  - System message: `You are an ecommerce marketing assistant.`
  - Output parser enabled
- **Key expressions or variables used:**
  - `{{ $json.customer.billing_address.billing_first_name }}`
  - `{{ $json.items[0].name }}`
  - `{{ $json.items[0].totals.total }}`
- **Input and output connections:**
  - Main input: `Cart is Still Abandoned?1`
  - AI language model input: `OpenAI Chat Model1`
  - Main output: `Extrect Email Subject & Body from AI1`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - If `items[0]` does not exist, the prompt expressions fail
  - If customer billing info is absent, personalization becomes blank or errors
  - The output parser assumption may not match the model output format
  - The prompt does not strictly enforce a `Subject:` first line, but the next node assumes that structure
- **Sub-workflow reference:** None

---

## 2.5 Email Formatting and Outbound Actions

### Overview
This block converts the AI text into structured email fields, sends the customer email, prepares summary fields for internal use, and posts a Slack notification.

### Nodes Involved
- Extrect Email Subject & Body from AI1
- Send Abandoned cart email to customer1
- Prepare recovery Data1
- Notify internal team on slack1
- Sticky Note9
- Sticky Note10
- Sticky Note8
- Sticky Note11

### Node Details

#### Extrect Email Subject & Body from AI1
- **Type and technical role:** `n8n-nodes-base.code` — parses AI output into `email_subject` and `email_body`
- **Configuration choices:**
  - JavaScript logic:
    - Reads `$json.output`
    - Splits by line breaks
    - Treats line 1 as subject, removing `Subject: `
    - Treats remaining lines as message body
- **Key expressions or variables used:**
  - `$json.output`
- **Input and output connections:**
  - Input: `Genrate Personalized Reminder Email1`
  - Outputs:
    - `Prepare recovery Data1`
    - `Send Abandoned cart email to customer1`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If AI output does not include a first line beginning with `Subject: `, the subject extraction becomes unreliable
  - If output is empty or not a string, code execution fails
  - If the AI returns a one-line response, body may be empty
- **Sub-workflow reference:** None

#### Send Abandoned cart email to customer1
- **Type and technical role:** `n8n-nodes-base.gmail` — sends the reminder email through Gmail
- **Configuration choices:**
  - Subject: `={{ $json.email_subject }}`
  - Message: `={{ $json.email_body }}`
  - Attribution disabled
  - Recipient is currently a placeholder: `{{Customer_email_here}}`
  - Uses Gmail OAuth2 credentials named `Gmail account 38`
- **Key expressions or variables used:**
  - `{{$json.email_subject}}`
  - `{{$json.email_body}}`
- **Input and output connections:**
  - Input: `Extrect Email Subject & Body from AI1`
  - Output: none
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Recipient is not dynamically mapped yet; this must be replaced with the actual customer email
  - Gmail OAuth token expiration or missing scopes
  - Gmail sending limits or account restrictions
  - HTML/plain-text formatting expectations may differ from actual content
- **Sub-workflow reference:** None

#### Prepare recovery Data1
- **Type and technical role:** `n8n-nodes-base.set` — builds a compact payload for internal notification
- **Configuration choices:**
  - Sets:
    - `Product_name` from `$('Cart is Still Abandoned?1').item.json.items[0].name`
    - `total` from `$('Cart is Still Abandoned?1').item.json.items[0].totals.total`
    - `customer_email` from `$('Cart is Still Abandoned?1').item.json.customer.billing_address.billing_email`
  - It references the IF node output directly rather than the immediately previous node
- **Key expressions or variables used:**
  - `{{ $('Cart is Still Abandoned?1').item.json.items[0].name }}`
  - `{{ $('Cart is Still Abandoned?1').item.json.items[0].totals.total }}`
  - `{{ $('Cart is Still Abandoned?1').item.json.customer.billing_address.billing_email }}`
- **Input and output connections:**
  - Input: `Extrect Email Subject & Body from AI1`
  - Output: `Notify internal team on slack1`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - If the IF node has no `items[0]`, expressions fail
  - Cross-node references can be harder to maintain if node names change
  - Customer email is prepared here but not used by the Gmail node in current form
- **Sub-workflow reference:** None

#### Notify internal team on slack1
- **Type and technical role:** `n8n-nodes-base.slack` — posts an internal notification to a Slack channel
- **Configuration choices:**
  - Sends message text:
    - Product name
    - Price/total
  - Destination:
    - Channel mode
    - Channel ID: `C09S57E2JQ2`
  - Workflow link inclusion disabled
  - Uses Slack credentials named `Slack account 15`
- **Key expressions or variables used:**
  - `{{ $json.Product_name }}`
  - `{{ $json.total }}`
- **Input and output connections:**
  - Input: `Prepare recovery Data1`
  - Output: none
- **Version-specific requirements:** Type version `2.3`
- **Edge cases or potential failure types:**
  - Invalid Slack token or missing chat:write permissions
  - Channel access missing for the connected Slack app
  - Message sends even if Gmail fails, because the branches are independent after the Code node
- **Sub-workflow reference:** None

#### Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the Gmail node’s purpose and message content
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note10
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents preparation of internal notification data and Slack notification behavior
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| OpenAI Chat Model1 | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides GPT-4.1-mini model to the AI agent |  | Genrate Personalized Reminder Email1 (AI model port) | ## Abandoned Cart Verification & AI Reminder Process  This part of the workflow waits for some time to give the customer a chance to complete checkout. After the wait, it checks the cart again to see if products are still there and payment is still pending. If the cart is still abandoned, AI generates a friendly and personalized reminder email using the customer and cart details. The AI message is then prepared in the correct format so an email can be sent to the customer and a notification can also be sent to the store owner. If the cart is already checked out, the workflow stops automatically. |
| Normalize Incoming Cart Data1 | `n8n-nodes-base.set` | Maps webhook payload into normalized cart fields | Receive Cart Event1 | Wait before checking cart status1 | This node cleans and organizes the incoming cart data. |
| Wait before checking cart status1 | `n8n-nodes-base.wait` | Delays execution before rechecking cart status | Normalize Incoming Cart Data1 | Recheck cart status from store1 | ## Abandoned Cart Verification & AI Reminder Process  This part of the workflow waits for some time to give the customer a chance to complete checkout. After the wait, it checks the cart again to see if products are still there and payment is still pending. If the cart is still abandoned, AI generates a friendly and personalized reminder email using the customer and cart details. The AI message is then prepared in the correct format so an email can be sent to the customer and a notification can also be sent to the store owner. If the cart is already checked out, the workflow stops automatically. |
| Recheck cart status from store1 | `n8n-nodes-base.httpRequest` | Calls store/cart API to retrieve latest cart state | Wait before checking cart status1 | Cart is Still Abandoned?1 | ## Abandoned Cart Verification & AI Reminder Process  This part of the workflow waits for some time to give the customer a chance to complete checkout. After the wait, it checks the cart again to see if products are still there and payment is still pending. If the cart is still abandoned, AI generates a friendly and personalized reminder email using the customer and cart details. The AI message is then prepared in the correct format so an email can be sent to the customer and a notification can also be sent to the store owner. If the cart is already checked out, the workflow stops automatically. |
| Cart is Still Abandoned?1 | `n8n-nodes-base.if` | Checks whether cart still has items and still needs payment | Recheck cart status from store1 | Genrate Personalized Reminder Email1 | ## Abandoned Cart Verification & AI Reminder Process  This part of the workflow waits for some time to give the customer a chance to complete checkout. After the wait, it checks the cart again to see if products are still there and payment is still pending. If the cart is still abandoned, AI generates a friendly and personalized reminder email using the customer and cart details. The AI message is then prepared in the correct format so an email can be sent to the customer and a notification can also be sent to the store owner. If the cart is already checked out, the workflow stops automatically. |
| Genrate Personalized Reminder Email1 | `@n8n/n8n-nodes-langchain.agent` | Generates personalized reminder email draft | Cart is Still Abandoned?1; OpenAI Chat Model1 | Extrect Email Subject & Body from AI1 | ## Abandoned Cart Verification & AI Reminder Process  This part of the workflow waits for some time to give the customer a chance to complete checkout. After the wait, it checks the cart again to see if products are still there and payment is still pending. If the cart is still abandoned, AI generates a friendly and personalized reminder email using the customer and cart details. The AI message is then prepared in the correct format so an email can be sent to the customer and a notification can also be sent to the store owner. If the cart is already checked out, the workflow stops automatically. |
| Extrect Email Subject & Body from AI1 | `n8n-nodes-base.code` | Parses AI output into subject and body | Genrate Personalized Reminder Email1 | Prepare recovery Data1; Send Abandoned cart email to customer1 | ## Abandoned Cart Verification & AI Reminder Process  This part of the workflow waits for some time to give the customer a chance to complete checkout. After the wait, it checks the cart again to see if products are still there and payment is still pending. If the cart is still abandoned, AI generates a friendly and personalized reminder email using the customer and cart details. The AI message is then prepared in the correct format so an email can be sent to the customer and a notification can also be sent to the store owner. If the cart is already checked out, the workflow stops automatically. |
| Prepare recovery Data1 | `n8n-nodes-base.set` | Prepares compact data for Slack notification | Extrect Email Subject & Body from AI1 | Notify internal team on slack1 | ## Prepare Data & Notify team  **Prepare Recovery Data:** This node prepares final data needed for notifications to internal team  **Notify internal team on slack:** This node sends a notification to the internal team on Slack. |
| Send Abandoned cart email to customer1 | `n8n-nodes-base.gmail` | Sends AI-generated reminder email to customer | Extrect Email Subject & Body from AI1 |  | This node sends the AI-generated reminder email to the customer.  The email includes: - Friendly message - Product details - Total price Goal: Encourage customer to complete checkout. |
| Notify internal team on slack1 | `n8n-nodes-base.slack` | Sends internal Slack alert about triggered recovery | Prepare recovery Data1 |  | ## Prepare Data & Notify team  **Prepare Recovery Data:** This node prepares final data needed for notifications to internal team  **Notify internal team on slack:** This node sends a notification to the internal team on Slack. |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Documents webhook trigger purpose |  |  |  |
| Receive Cart Event1 | `n8n-nodes-base.webhook` | Receives cart event from store |  | Normalize Incoming Cart Data1 | When a user adds products to the cart and leaves the website without checkout, the website sends cart details (cart key, email, name, etc.) to this webhook. |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Documents normalization step |  |  |  |
| Sticky Note8 | `n8n-nodes-base.stickyNote` | Documents verification and AI reminder block |  |  |  |
| Sticky Note9 | `n8n-nodes-base.stickyNote` | Documents Gmail send step |  |  |  |
| Sticky Note10 | `n8n-nodes-base.stickyNote` | Documents Slack preparation and notification step |  |  |  |
| Sticky Note11 | `n8n-nodes-base.stickyNote` | Documents full workflow behavior and setup steps |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## 4.1 Create the entry webhook
1. Add a **Webhook** node.
2. Name it **Receive Cart Event1**.
3. Set the **Path** to `abandoned-cart`.
4. Keep the method and options as needed by your store integration, typically `POST`.
5. Save the node.
6. Make sure your store will send a payload with fields similar to:
   - `body.cart_key`
   - `body.email`
   - `body.first_name`
   - `body.cart_total`
   - `body.item_name`
   - `body.item_quantity`

## 4.2 Normalize incoming cart data
7. Add a **Set** node.
8. Name it **Normalize Incoming Cart Data1**.
9. Connect **Receive Cart Event1 → Normalize Incoming Cart Data1**.
10. Add fields:
    - `cart_key` = `{{$json.body.cart_key}}`
    - `email` = `{{$json.body.email}}`
    - `first_name` = `{{$json.body.first_name}}`
    - `cart_total` = `{{$json.body.cart_total}}`
    - `item_name` = `{{$json.body.item_name}}`
    - `item_quantity` = `{{$json.body.item_quantity}}`
11. Important: do **not** reproduce the accidental field name `=item_name`; use `item_name`.

## 4.3 Add the waiting period
12. Add a **Wait** node.
13. Name it **Wait before checking cart status1**.
14. Connect **Normalize Incoming Cart Data1 → Wait before checking cart status1**.
15. Configure the delay explicitly, for example:
    - 30 minutes
    - or 60 minutes
16. Ensure your n8n instance supports paused/resumable executions.

## 4.4 Recheck the cart from WooCommerce/CoCart
17. Add an **HTTP Request** node.
18. Name it **Recheck cart status from store1**.
19. Connect **Wait before checking cart status1 → Recheck cart status from store1**.
20. Set the request URL to your real cart lookup endpoint, for example your CoCart cart endpoint.
21. Configure the request to send a header:
    - Header name: `cocart-cart-key`
    - Header value: `{{$json.cart_key}}`
22. If your API requires auth, configure the correct authentication.
23. Ensure the response contains fields such as:
    - `item_count`
    - `needs_payment`
    - `items`
    - `customer.billing_address.billing_first_name`
    - `customer.billing_address.billing_email`
24. Important: remove the leading space from the header name if you are rebuilding manually.

## 4.5 Add the abandonment check
25. Add an **IF** node.
26. Name it **Cart is Still Abandoned?1**.
27. Connect **Recheck cart status from store1 → Cart is Still Abandoned?1**.
28. Configure the IF node with **AND** logic.
29. Add condition 1:
    - Left value: `{{$json.item_count}}`
    - Operator: **greater than**
    - Right value: `0`
30. Add condition 2:
    - Left value: `{{$json.needs_payment}}`
    - Operator: **is true**
31. Leave the false branch unconnected if you want the workflow to stop silently when the cart is no longer abandoned.

## 4.6 Add the OpenAI model
32. Add an **OpenAI Chat Model** node from the LangChain/OpenAI nodes.
33. Name it **OpenAI Chat Model1**.
34. Select model **gpt-4.1-mini**.
35. Connect OpenAI credentials.
36. Use a valid OpenAI API credential in n8n.

## 4.7 Add the AI Agent
37. Add an **AI Agent** node.
38. Name it **Genrate Personalized Reminder Email1**.
39. Connect the **true** output of **Cart is Still Abandoned?1** to this agent.
40. Connect **OpenAI Chat Model1** to the agent’s **AI language model** input.
41. Set prompt type to **Define below** or equivalent manual prompt mode.
42. Use a prompt similar to:

    Create a personalized abandoned cart email.  
    Customer name: `{{ $json.customer.billing_address.billing_first_name }}`  
    Product: `{{ $json.items[0].name }}`  
    Total: `₹{{ $json.items[0].totals.total }}`  
    Keep it under 120 words.

43. Set the system message to:
    - `You are an ecommerce marketing assistant.`
44. Enable output parsing if desired.
45. Best practice: instruct the model to always output in this exact structure:
    - First line: `Subject: ...`
    - Remaining lines: body  
   This makes the next Code node much safer.

## 4.8 Extract subject and body
46. Add a **Code** node.
47. Name it **Extrect Email Subject & Body from AI1**.
48. Connect **Genrate Personalized Reminder Email1 → Extrect Email Subject & Body from AI1**.
49. Use JavaScript that:
    - reads the AI result from `$json.output`
    - splits lines
    - converts the first line into `email_subject`
    - joins the remaining lines into `email_body`
50. Example logic:
    - `const aiOutput = $json.output;`
    - split by newline
    - strip `Subject: `
    - return `{ email_subject, email_body }`

## 4.9 Send the Gmail reminder
51. Add a **Gmail** node.
52. Name it **Send Abandoned cart email to customer1**.
53. Connect **Extrect Email Subject & Body from AI1 → Send Abandoned cart email to customer1**.
54. Configure Gmail OAuth2 credentials.
55. Set:
    - **To** = ideally the real customer email, such as `{{$('Cart is Still Abandoned?1').item.json.customer.billing_address.billing_email}}`
    - **Subject** = `{{$json.email_subject}}`
    - **Message** = `{{$json.email_body}}`
56. Disable attribution if you do not want n8n branding appended.
57. Important: the exported workflow still contains placeholder text `{{Customer_email_here}}`; replace it with a live expression.

## 4.10 Prepare Slack notification data
58. Add another **Set** node.
59. Name it **Prepare recovery Data1**.
60. Connect **Extrect Email Subject & Body from AI1 → Prepare recovery Data1**.
61. Add fields:
    - `Product_name` = `{{ $('Cart is Still Abandoned?1').item.json.items[0].name }}`
    - `total` = `{{ $('Cart is Still Abandoned?1').item.json.items[0].totals.total }}`
    - `customer_email` = `{{ $('Cart is Still Abandoned?1').item.json.customer.billing_address.billing_email }}`
62. This node is only for preparing internal notification data.

## 4.11 Notify Slack
63. Add a **Slack** node.
64. Name it **Notify internal team on slack1**.
65. Connect **Prepare recovery Data1 → Notify internal team on slack1**.
66. Configure Slack credentials with permission to post messages.
67. Choose **Channel** mode.
68. Select the destination channel.
69. Set the message text to something like:

    This Product is in User's cart:

    Product Name: `{{ $json.Product_name }}`
    Price: `{{ $json.total }}`

70. Optionally include customer email too for internal follow-up.

## 4.12 Add documentation sticky notes
71. Add a **Sticky Note** near the webhook explaining that the store sends abandoned-cart details here.
72. Add a second **Sticky Note** near the Set node explaining that it normalizes incoming data.
73. Add a large **Sticky Note** around the wait / verification / AI section describing the full recovery process.
74. Add a **Sticky Note** near Gmail explaining that the node sends the AI-generated reminder email.
75. Add a **Sticky Note** near the Set + Slack nodes explaining internal-team notification behavior.
76. Optionally add a large top-level **Sticky Note** with setup instructions and the overall workflow logic.

## 4.13 Credential setup checklist
77. **OpenAI credential**
    - Create or select an OpenAI API credential
    - Ensure access to `gpt-4.1-mini`
78. **Gmail OAuth2 credential**
    - Connect a Gmail account with send permissions
    - Verify OAuth scopes are sufficient
79. **Slack credential**
    - Connect a Slack app or bot token
    - Confirm it can post to the target channel
80. **Store/cart API**
    - Ensure the API endpoint is reachable from n8n
    - If using CoCart or WooCommerce-related API, verify header and auth requirements

## 4.14 Recommended corrections before production use
81. Replace all placeholders:
    - `{{your_cart_API_here}}`
    - `{{Customer_email_here}}`
82. Correct the normalized field name from `=item_name` to `item_name`.
83. Correct the HTTP header name from ` cocart-cart-key` to `cocart-cart-key`.
84. Explicitly configure the Wait node duration.
85. Strengthen the AI prompt so the output format is deterministic.
86. Consider using the normalized email directly if the recheck API does not always return billing email.
87. Consider error handling branches for:
    - API failure
    - AI failure
    - Gmail failure
    - Slack failure

## 4.15 No sub-workflows
88. This workflow does **not** invoke any sub-workflow nodes.
89. There are no Execute Workflow nodes or external n8n workflow dependencies.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| When a customer leaves items in their cart, the store sends cart details to this workflow using a webhook. The workflow saves the cart information and waits for a short period to give the customer time to complete checkout. After waiting, it checks the cart again from the store. If the cart still has products and payment is not completed, the workflow treats it as an abandoned cart. An AI assistant then creates a friendly, personalized reminder email using the customer name, product name, and cart total. The email content is prepared properly and sent to the customer. At the same time, a notification is sent to Slack so the team knows a reminder was triggered. If the customer has already completed the purchase, the workflow stops automatically. | General workflow behavior |
| Setup steps included in the workflow notes: connect the webhook to the store, configure wait time, configure the cart API, connect OpenAI, set up Gmail, and connect Slack. | Operational setup guidance |
| No external links are embedded in the sticky notes of this workflow. | Documentation review |
| The workflow title is: Recover abandoned WooCommerce carts using OpenAI GPT-4.1-mini, Gmail and Slack. | Project identification |

## Final implementation cautions
- The workflow is structurally valid but contains placeholders and a few configuration issues that must be fixed before production.
- The Gmail node currently does not use the customer email dynamically.
- The AI parsing logic depends on the model returning a first line beginning with `Subject:`.
- The API response schema is critical; downstream nodes assume a WooCommerce/CoCart-like JSON structure.