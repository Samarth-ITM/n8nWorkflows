Automate service order fulfillment with Claude, Freshworks CRM and SLA escalation

https://n8nworkflows.xyz/workflows/automate-service-order-fulfillment-with-claude--freshworks-crm-and-sla-escalation-14164


# Automate service order fulfillment with Claude, Freshworks CRM and SLA escalation

## 1. Workflow Overview

This workflow automates service order fulfillment for operational brokerages such as skip hire, field services, or logistics coordination. It accepts a paid order from an external checkout/backend, validates the payload, verifies payment with Stripe, uses Claude to structure the free-text order data, creates customer and deal records in Freshworks CRM, sends transactional emails via Postmark, assigns a supplier, enforces SLA windows with wait-and-check logic, escalates missed SLAs to Slack, retries with another supplier, and logs all final outcomes to Google Sheets.

The workflow is designed for:
- Paid service order intake
- CRM-driven dispatch operations
- Supplier assignment with SLA monitoring
- Escalation handling when suppliers do not respond
- Outcome tracking for reporting and auditability

### 1.1 Input Reception and Validation
The workflow starts from a webhook and normalizes incoming order data. It checks required fields, validates email format, generates an order reference if needed, and standardizes values such as lowercase email and uppercase postcode.

### 1.2 Payment Verification
The normalized order is checked against Stripe using the `payment_intent_id`. The result is consolidated into a single operational payload and routed depending on whether payment succeeded.

### 1.3 Payment Failure Handling
If payment is not successful, the workflow sends a failed-payment notice and logs the failure to Google Sheets. Processing stops on this branch.

### 1.4 AI-Based Order Structuring
If payment is valid, Claude parses the unstructured order notes into structured operational fields such as waste type, skip size, restrictions, and urgency using a structured output schema.

### 1.5 CRM Customer and Deal Creation
The workflow upserts the customer into Freshworks CRM, then creates a service deal associated with the order context.

### 1.6 Customer Confirmation and Initial Supplier Assignment
A confirmation email is sent to the customer. The workflow then looks up available suppliers in Freshworks, selects the first candidate, and sends them an assignment request.

### 1.7 First SLA Monitoring
After assignment, the workflow waits for the first SLA window of 4 hours, then checks the Freshworks deal stage to determine whether the supplier accepted.

### 1.8 Success Path After First SLA
If the supplier accepted within the first SLA window, the deal is updated, the customer is notified, and the success is logged to Google Sheets.

### 1.9 Escalation and Retry Assignment
If the first SLA is missed, the workflow alerts Slack, selects another supplier, sends a reassignment request, and starts a second SLA window of 2 hours.

### 1.10 Final Retry Decision and Manual Escalation
After the retry window, the deal stage is checked again. If accepted, the workflow confirms and logs success. If not, it raises a critical Slack alert, marks the deal as escalated, and logs the escalation.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

**Overview:**  
This block receives a new service order via webhook and converts it into a consistent internal structure. It rejects incomplete or malformed payloads early before any external systems are called.

**Nodes Involved:**  
- New Order Webhook
- Validate & Parse Order

### Node Details

#### New Order Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point for external HTTP POST requests.
- **Configuration choices:**  
  - Path: `new-order`
  - Method: `POST`
  - Response mode: `responseNode`
  - Raw body disabled
- **Key expressions or variables used:** None directly.
- **Input and output connections:**  
  - No input; entry node
  - Outputs to `Validate & Parse Order`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - External system posts invalid JSON or wrong content type
  - Because response mode is `responseNode` but no response node exists, this may require attention depending on n8n version/runtime behavior
  - Webhook URL must be correctly deployed in production mode
- **Sub-workflow reference:** None

#### Validate & Parse Order
- **Type and technical role:** `n8n-nodes-base.code`; validates input fields and normalizes order values.
- **Configuration choices:**  
  - Reads either `json.body` or root `json`
  - Requires:
    - `payment_intent_id`
    - `customer_email`
    - `order_notes`
  - Lowercases email
  - Generates `order_ref` as `ORD-${Date.now()}` if missing
  - Uppercases `postcode`
  - Parses `order_value_gbp` as float
  - Adds `received_at`
- **Key expressions or variables used:**  
  - `const item = $input.first().json?.body || $input.first().json;`
  - `missing = required.filter(...)`
  - `orderRef = item.order_ref || \`ORD-${Date.now()}\``
- **Input and output connections:**  
  - Input from `New Order Webhook`
  - Output to `Verify Stripe Payment`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - Missing required fields throws an error
  - Invalid email format throws an error
  - `order_value_gbp` defaults to `0` if not numeric
  - Empty optional fields become empty strings, which may later reduce CRM/email quality
- **Sub-workflow reference:** None

---

## 2.2 Payment Verification

**Overview:**  
This block verifies the Stripe payment intent and merges payment data back into the order context. It determines whether the order is allowed to continue.

**Nodes Involved:**  
- Verify Stripe Payment
- Consolidate Payment Result
- Payment Succeeded?

### Node Details

#### Verify Stripe Payment
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls Stripe Payment Intents API.
- **Configuration choices:**  
  - GET request to `https://api.stripe.com/v1/payment_intents/{{ $json.payment_intent_id }}`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Stripe-Version: 2024-11-20.acacia`
  - `neverError: true`
  - Retries enabled: 3 tries, 2-second wait
  - `onError: continueErrorOutput`
- **Key expressions or variables used:**  
  - URL expression uses `{{$json.payment_intent_id}}`
- **Input and output connections:**  
  - Input from `Validate & Parse Order`
  - Output to `Consolidate Payment Result`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**  
  - Invalid or expired Stripe secret key
  - Wrong API version compatibility
  - Payment intent not found
  - Non-200 responses are tolerated because `neverError` is enabled, so downstream code must interpret missing/partial data correctly
- **Sub-workflow reference:** None

#### Consolidate Payment Result
- **Type and technical role:** `n8n-nodes-base.code`; merges Stripe response with normalized order data.
- **Configuration choices:**  
  - Reads original order from `Validate & Parse Order`
  - Reads Stripe result from current input
  - Maps:
    - `payment_status`
    - `payment_valid`
    - `payment_amount_gbp`
    - `payment_currency`
    - `stripe_customer_id`
    - `payment_error`
- **Key expressions or variables used:**  
  - `$('Validate & Parse Order').first().json`
  - `payment.status || 'error'`
  - `status === 'succeeded'`
  - `amountPence / 100`
- **Input and output connections:**  
  - Input from `Verify Stripe Payment`
  - Output to `Payment Succeeded?`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - If Stripe returns an error payload with no `status`, the workflow treats it as `error`
  - Amount defaults to `0`
  - If the upstream node returns malformed data, fields may be blank and cause false negatives
- **Sub-workflow reference:** None

#### Payment Succeeded?
- **Type and technical role:** `n8n-nodes-base.if`; routes based on `payment_valid`.
- **Configuration choices:**  
  - Loose type validation
  - Boolean equality check: `{{$json.payment_valid}} == true`
- **Key expressions or variables used:**  
  - `={{ $json.payment_valid }}`
- **Input and output connections:**  
  - Input from `Consolidate Payment Result`
  - True branch to `Extract Order Details`
  - False branch to `Send Payment Failed Notice`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**  
  - Loose validation is forgiving, but unexpected non-boolean values may still create ambiguous behavior
- **Sub-workflow reference:** None

---

## 2.3 Payment Failure Handling

**Overview:**  
This block handles orders whose Stripe payment did not succeed. It notifies the customer and records the failure in the fulfillment log.

**Nodes Involved:**  
- Send Payment Failed Notice
- Log Failed Payment

### Node Details

#### Send Payment Failed Notice
- **Type and technical role:** `n8n-nodes-base.httpRequest`; intended to send an email via Postmark.
- **Configuration choices:**  
  - POST to `https://api.postmarkapp.com/email`
  - Headers:
    - `X-Postmark-Server-Token: YOUR_POSTMARK_SERVER_TOKEN`
    - `Accept: application/json`
  - Body sending enabled, but the body parameter list is effectively empty
  - `neverError: true`
  - Retries enabled
- **Key expressions or variables used:** None currently configured in body.
- **Input and output connections:**  
  - Input from false branch of `Payment Succeeded?`
  - Output to `Log Failed Payment`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**  
  - As configured, this node is incomplete: Postmark requires fields such as `From`, `To`, `Subject`, and body content
  - Invalid Postmark server token
  - Because `neverError` is enabled, failed delivery may still pass downstream
- **Sub-workflow reference:** None

#### Log Failed Payment
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends or updates a failure record.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Document ID placeholder: `YOUR_FULFILLMENT_SHEET_ID`
  - Sheet name: `Order Log`
  - Column mapping:
    - Reason
    - Status = `payment_failed`
    - Customer
    - Order Ref
    - Timestamp
    - Order Value
  - Uses Google Sheets OAuth2 credentials
  - Retries enabled
- **Key expressions or variables used:**  
  - `={{ $json.payment_error || 'payment not succeeded' }}`
  - `={{ DateTime.now().toISO() }}`
- **Input and output connections:**  
  - Input from `Send Payment Failed Notice`
  - No downstream node
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**  
  - Wrong spreadsheet ID or missing tab
  - Append/update behavior depends on sheet configuration and matching rules
  - `DateTime` assumes Luxon availability in n8n expressions
- **Sub-workflow reference:** None

---

## 2.4 AI-Based Order Structuring

**Overview:**  
This block uses Claude with a structured output parser to transform free-text customer order notes into operationally useful fields. It is central to turning loosely formatted checkout data into CRM-ready attributes.

**Nodes Involved:**  
- Extract Order Details
- Claude — Extract
- Order Details Schema

### Node Details

#### Extract Order Details
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI agent orchestrating prompt + model + output parser.
- **Configuration choices:**  
  - Prompt includes:
    - order reference
    - service type
    - collection address
    - postcode
    - requested date
    - free-text notes
    - estimated order value
  - System message enforces:
    - high-precision extraction
    - skip-size recognition
    - waste-type classification
    - access restriction extraction
    - urgency normalization
    - always output all fields
  - `hasOutputParser: true`
  - No intermediate steps returned
- **Key expressions or variables used:**  
  - `{{ $json.order_ref }}`
  - `{{ $json.service_type }}`
  - `{{ $json.collection_address }}`
  - `{{ $json.postcode }}`
  - `{{ $json.requested_date }}`
  - `{{ $json.order_notes }}`
  - `{{ $json.order_value_gbp }}`
- **Input and output connections:**  
  - Main input from true branch of `Payment Succeeded?`
  - AI language model input from `Claude — Extract`
  - AI output parser input from `Order Details Schema`
  - Main output to `Upsert Customer Contact`
- **Version-specific requirements:** Type version `1.8`; requires LangChain-compatible n8n installation and Anthropic support.
- **Edge cases or potential failure types:**  
  - AI may infer fields incorrectly from sparse notes
  - Structured parser can fail if output violates schema
  - Required schema fields can force null/empty synthesis if source data is poor
  - Anthropic credential or quota failures
- **Sub-workflow reference:** None

#### Claude — Extract
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Anthropic chat model provider.
- **Configuration choices:**  
  - Model: `claude-3-5-sonnet-20241022`
  - Temperature: `0`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connects to `Extract Order Details` via AI language model port
- **Version-specific requirements:** Type version `1.3`; requires Anthropic credentials and model availability.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Model deprecation or regional availability limits
  - Rate limits / token limits
- **Sub-workflow reference:** None

#### Order Details Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces JSON schema output.
- **Configuration choices:**  
  - Manual schema with fields such as:
    - `service_type`
    - `collection_address`
    - `postcode`
    - `waste_type`
    - `skip_size`
    - `requested_date`
    - `customer_name`
    - `customer_email`
    - `customer_phone`
    - `special_instructions`
    - `access_restrictions`
    - `estimated_value_gbp`
    - `priority`
  - Required fields include:
    - `service_type`
    - `collection_address`
    - `postcode`
    - `waste_type`
    - `customer_name`
    - `customer_email`
    - `customer_phone`
    - `priority`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connects to `Extract Order Details` via AI output parser port
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**  
  - Schema mismatch can stop the AI block
  - Enum mismatches for `waste_type` or `priority`
- **Sub-workflow reference:** None

---

## 2.5 CRM Customer and Deal Creation

**Overview:**  
This block writes the customer and order into Freshworks CRM. It first upserts the contact through a custom REST call, then creates the service deal using the Freshworks CRM node.

**Nodes Involved:**  
- Upsert Customer Contact
- Consolidate Contact
- Create Service Deal
- Consolidate Deal

### Node Details

#### Upsert Customer Contact
- **Type and technical role:** `n8n-nodes-base.httpRequest`; custom Freshworks contact upsert call.
- **Configuration choices:**  
  - POST to `https://YOUR_DOMAIN.freshsales.io/api/upsert/contacts`
  - Headers:
    - `Authorization: Token token=YOUR_FRESHWORKS_API_KEY`
    - `Content-Type: application/json`
  - Body sending enabled, but currently empty
  - `neverError: true`
  - Retries enabled
- **Key expressions or variables used:** None currently in body.
- **Input and output connections:**  
  - Input from `Extract Order Details`
  - Output to `Consolidate Contact`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**  
  - As configured, body payload is incomplete, so true upsert will not work until contact fields are added
  - Incorrect domain or API key
  - Freshworks endpoint availability depends on plan/API behavior
- **Sub-workflow reference:** None

#### Consolidate Contact
- **Type and technical role:** `n8n-nodes-base.code`; merges order, payment, AI extraction, and contact response.
- **Configuration choices:**  
  - Pulls data from:
    - `Validate & Parse Order`
    - `Consolidate Payment Result`
    - `Extract Order Details`
    - current Freshworks upsert response
  - Produces:
    - `contact_id`
    - normalized customer identity
    - address and postcode
    - `waste_type`
    - `skip_size`
    - `priority`
    - `special_instructions`
    - `access_restrictions`
- **Key expressions or variables used:**  
  - `$('Extract Order Details').first().json.output || {}`
  - `String(contact.id || contact.contact?.id || '')`
- **Input and output connections:**  
  - Input from `Upsert Customer Contact`
  - Output to `Create Service Deal`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - If AI output is absent, the node falls back to original order values where possible
  - If contact response format differs, `contact_id` may end up blank
- **Sub-workflow reference:** None

#### Create Service Deal
- **Type and technical role:** `n8n-nodes-base.freshworksCrm`; creates a deal.
- **Configuration choices:**  
  - Resource: `deal`
  - Name expression: `{{ $json.service_type | upper }} — {{ $json.postcode }} — {{ $json.order_ref }}`
  - Additional field:
    - `probability: 100`
  - Retries enabled
- **Key expressions or variables used:**  
  - `={{ $json.service_type | upper }} — {{ $json.postcode }} — {{ $json.order_ref }}`
- **Input and output connections:**  
  - Input from `Consolidate Contact`
  - Output to `Consolidate Deal`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - Node appears underconfigured: likely needs stage, owner, currency, value, and association to contact depending on desired CRM behavior
  - Expression filter `| upper` may depend on expression support; if unsupported in a given context, deal naming may fail
  - Missing Freshworks credential configuration
- **Sub-workflow reference:** None

#### Consolidate Deal
- **Type and technical role:** `n8n-nodes-base.code`; stores the created deal ID and name in workflow context.
- **Configuration choices:**  
  - Reads current deal result
  - Reads prior context from `Consolidate Contact`
  - Extracts:
    - `deal_id`
    - `deal_name`
- **Key expressions or variables used:**  
  - `String(deal.id || deal.deal?.id || '')`
- **Input and output connections:**  
  - Input from `Create Service Deal`
  - Output to `Send Order Confirmation`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - If Freshworks returns an unexpected shape, `deal_id` may be blank and break downstream deal checks/updates
- **Sub-workflow reference:** None

---

## 2.6 Customer Confirmation and Initial Supplier Assignment

**Overview:**  
This block sends the initial customer confirmation, looks up candidate suppliers in Freshworks, picks the first available result, and sends the assignment email.

**Nodes Involved:**  
- Send Order Confirmation
- Find Available Supplier
- Consolidate Supplier
- Send Supplier Assignment

### Node Details

#### Send Order Confirmation
- **Type and technical role:** `n8n-nodes-base.httpRequest`; intended Postmark customer email.
- **Configuration choices:**  
  - POST to Postmark email API
  - Header includes server token and `Accept: application/json`
  - Body is enabled but empty
  - `neverError: true`
- **Key expressions or variables used:** None currently in body.
- **Input and output connections:**  
  - Input from `Consolidate Deal`
  - Output to `Find Available Supplier`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**  
  - Currently incomplete for real email sending
  - Could silently fail because errors are suppressed
- **Sub-workflow reference:** None

#### Find Available Supplier
- **Type and technical role:** `n8n-nodes-base.freshworksCrm`; fetches contacts from Freshworks.
- **Configuration choices:**  
  - Resource: `contact`
  - Operation: `getAll`
  - Limit: `5`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from `Send Order Confirmation`
  - Output to `Consolidate Supplier`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - The sticky note says suppliers should be tagged `supplier-available`, but the node has no visible filter configured
  - Without filtering, it may return arbitrary contacts rather than suppliers
  - Paging/ordering may affect which “first” supplier is chosen
- **Sub-workflow reference:** None

#### Consolidate Supplier
- **Type and technical role:** `n8n-nodes-base.code`; chooses the first supplier candidate and captures fallback state if none exist.
- **Configuration choices:**  
  - Reads all returned contacts
  - Filters out rows with errors and rows lacking `id`
  - If none found:
    - sets `supplier_found = false`
    - sets placeholder supplier fields
  - Otherwise:
    - selects first contact as primary supplier
    - stores all supplier IDs in `all_supplier_ids`
- **Key expressions or variables used:**  
  - `rows.filter(r => !r.json.error && r.json.id)`
  - `suppliers[0].json`
- **Input and output connections:**  
  - Input from `Find Available Supplier`
  - Output to `Send Supplier Assignment`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - No supplier found still proceeds downstream
  - If first contact has no email, supplier notification may fail
  - No logic uses tags, geography, service fit, or availability score
- **Sub-workflow reference:** None

#### Send Supplier Assignment
- **Type and technical role:** `n8n-nodes-base.httpRequest`; intended Postmark assignment email to supplier.
- **Configuration choices:**  
  - POST to Postmark email API
  - Header includes server token
  - Body is enabled but empty
  - `neverError: true`
- **Key expressions or variables used:** None currently in body.
- **Input and output connections:**  
  - Input from `Consolidate Supplier`
  - Output to `SLA Wait — 4 Hours`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**  
  - Incomplete email configuration
  - If `supplier_found` is false, the node still runs unless additional logic is added
- **Sub-workflow reference:** None

---

## 2.7 First SLA Monitoring

**Overview:**  
This block pauses execution for the main SLA window, checks the current deal stage in Freshworks, and determines whether acceptance occurred.

**Nodes Involved:**  
- SLA Wait — 4 Hours
- Check Deal Stage
- Consolidate SLA Status
- Supplier Accepted SLA?

### Node Details

#### SLA Wait — 4 Hours
- **Type and technical role:** `n8n-nodes-base.wait`; delays execution until the SLA window expires.
- **Configuration choices:**  
  - Wait node present, but no explicit wait duration is configured in the provided JSON
  - Has webhook ID `sla-wait-4h-uuid-002`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from `Send Supplier Assignment`
  - Output to `Check Deal Stage`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**  
  - Despite the name/sticky note, the actual wait duration is not configured in the JSON
  - On rebuild, duration must be explicitly set to 4 hours
  - Wait nodes require persistent execution storage in production
- **Sub-workflow reference:** None

#### Check Deal Stage
- **Type and technical role:** `n8n-nodes-base.freshworksCrm`; fetches current deal details.
- **Configuration choices:**  
  - Resource: `deal`
  - Operation: `get`
  - Deal ID from `{{$json.deal_id}}`
- **Key expressions or variables used:**  
  - `={{ $json.deal_id }}`
- **Input and output connections:**  
  - Input from `SLA Wait — 4 Hours`
  - Output to `Consolidate SLA Status`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - Blank `deal_id` causes lookup failure
  - Freshworks latency or permission issues
- **Sub-workflow reference:** None

#### Consolidate SLA Status
- **Type and technical role:** `n8n-nodes-base.code`; interprets the deal stage as accepted or not.
- **Configuration choices:**  
  - Reads stage from:
    - `deal.deal_stage.name`
    - or `deal.stage_name`
  - Lowercases stage
  - Matches accepted stages:
    - accepted
    - confirmed
    - partner confirmed
    - supplier accepted
    - in progress
  - Sets `sla_attempt = 1`
- **Key expressions or variables used:**  
  - `acceptedStages.some(s => stageName.includes(s))`
- **Input and output connections:**  
  - Input from `Check Deal Stage`
  - Output to `Supplier Accepted SLA?`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - Custom deal stage names in Freshworks must align with this logic
  - Partial string matching may create false positives/negatives
- **Sub-workflow reference:** None

#### Supplier Accepted SLA?
- **Type and technical role:** `n8n-nodes-base.if`; routes accepted vs escalation path.
- **Configuration choices:**  
  - Boolean equality check on `supplier_accepted`
  - Loose type validation
- **Key expressions or variables used:**  
  - `={{ $json.supplier_accepted }}`
- **Input and output connections:**  
  - Input from `Consolidate SLA Status`
  - True branch to `Update Deal — Confirmed`
  - False branch to `Alert Slack — SLA Missed`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**  
  - Incorrect upstream parsing of stage name affects branch routing
- **Sub-workflow reference:** None

---

## 2.8 Success Path After First SLA

**Overview:**  
If the supplier accepted in time, this block confirms the deal state, informs the customer, and logs the successful fulfillment outcome.

**Nodes Involved:**  
- Update Deal — Confirmed
- Send Customer — Job Confirmed
- Log Success

### Node Details

#### Update Deal — Confirmed
- **Type and technical role:** `n8n-nodes-base.freshworksCrm`; updates the deal after supplier acceptance.
- **Configuration choices:**  
  - Resource: `deal`
  - Operation: `update`
  - Deal ID from `{{$json.deal_id}}`
  - `updateFields` currently empty
- **Key expressions or variables used:**  
  - `={{ $json.deal_id }}`
- **Input and output connections:**  
  - Input from true branch of `Supplier Accepted SLA?`
  - Output to `Send Customer — Job Confirmed`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - As configured, this likely performs no meaningful stage update because `updateFields` is empty
  - Must be completed with actual confirmed stage/status mapping
- **Sub-workflow reference:** None

#### Send Customer — Job Confirmed
- **Type and technical role:** `n8n-nodes-base.httpRequest`; intended Postmark confirmation email.
- **Configuration choices:**  
  - POST to Postmark
  - Body enabled but empty
  - `neverError: true`
- **Key expressions or variables used:** None currently in body.
- **Input and output connections:**  
  - Input from `Update Deal — Confirmed`
  - Output to `Log Success`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**  
  - Incomplete body configuration
  - Can continue despite failed message delivery
- **Sub-workflow reference:** None

#### Log Success
- **Type and technical role:** `n8n-nodes-base.googleSheets`; logs first-attempt success.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Sheet: `Order Log`
  - Status: `confirmed`
  - SLA Attempt: `1`
  - Columns include deal, service, customer, postcode, supplier, order ref, skip size, waste type, order value, timestamp
- **Key expressions or variables used:**  
  - Extensive references to `$('Consolidate SLA Status').first().json...`
  - `={{ DateTime.now().toISO() }}`
- **Input and output connections:**  
  - Input from `Send Customer — Job Confirmed`
  - No downstream node
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**  
  - Sheet schema mismatch
  - Duplicate handling depends on `appendOrUpdate` matching behavior
- **Sub-workflow reference:** None

---

## 2.9 Escalation and Retry Assignment

**Overview:**  
If the first SLA is missed, this block alerts operations in Slack, chooses another supplier, sends reassignment, and starts a shorter retry SLA window.

**Nodes Involved:**  
- Alert Slack — SLA Missed
- Find Next Supplier
- Consolidate Next Supplier
- Send Reassignment Request
- Retry Wait — 2 Hours

### Node Details

#### Alert Slack — SLA Missed
- **Type and technical role:** `n8n-nodes-base.slack`; sends an escalation message to operations.
- **Configuration choices:**  
  - Message text contains order, customer, service, address, priority, assigned supplier, order value, and deal ID
  - Intended channel mentioned in notes, but explicit channel field is not visible here
- **Key expressions or variables used:**  
  - `{{ $json.order_ref }}`
  - `{{ $json.customer_name }}`
  - `{{ $json.customer_email }}`
  - `{{ $json.service_type }}`
  - `{{ $json.skip_size || 'TBC' }}`
  - `{{ $json.collection_address }}`
  - `{{ $json.postcode }}`
  - `{{ $json.priority | upper }}`
  - `{{ $json.supplier_name }}`
  - `{{ $json.supplier_email }}`
  - `{{ $json.payment_amount_gbp }}`
  - `{{ $json.deal_id }}`
- **Input and output connections:**  
  - Input from false branch of `Supplier Accepted SLA?`
  - Output to `Find Next Supplier`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**  
  - Slack credential missing or unauthorized
  - Channel routing must be configured in node/credential settings
  - Expression filter `| upper` may not behave identically in all contexts
- **Sub-workflow reference:** None

#### Find Next Supplier
- **Type and technical role:** `n8n-nodes-base.freshworksCrm`; retrieves more contact candidates for reassignment.
- **Configuration choices:**  
  - Resource: `contact`
  - Operation: `getAll`
  - Limit: `10`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from `Alert Slack — SLA Missed`
  - Output to `Consolidate Next Supplier`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - Like the first supplier search, there is no visible supplier tag filter configured
  - May return non-supplier contacts
- **Sub-workflow reference:** None

#### Consolidate Next Supplier
- **Type and technical role:** `n8n-nodes-base.code`; chooses a supplier different from the original one.
- **Configuration choices:**  
  - Reads previous supplier ID from `Consolidate SLA Status`
  - Excludes that supplier from candidate list
  - If none left:
    - marks `next_supplier_found = false`
  - Otherwise stores next supplier identity
- **Key expressions or variables used:**  
  - `const firstSupplierId = ctx.supplier_id;`
  - `String(r.json.id) !== firstSupplierId`
- **Input and output connections:**  
  - Input from `Find Next Supplier`
  - Output to `Send Reassignment Request`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - If all available contacts include the original supplier only, retry path proceeds with no valid supplier
  - No service-area matching or capability filter
- **Sub-workflow reference:** None

#### Send Reassignment Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`; intended Postmark reassignment email.
- **Configuration choices:**  
  - POST to Postmark
  - Body enabled but empty
  - `neverError: true`
- **Key expressions or variables used:** None currently in body.
- **Input and output connections:**  
  - Input from `Consolidate Next Supplier`
  - Output to `Retry Wait — 2 Hours`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**  
  - Incomplete email payload
  - Still runs if `next_supplier_found` is false unless additional guard logic is added
- **Sub-workflow reference:** None

#### Retry Wait — 2 Hours
- **Type and technical role:** `n8n-nodes-base.wait`; pauses execution for retry SLA.
- **Configuration choices:**  
  - Node name indicates 2-hour wait
  - Actual wait duration is not configured in the JSON
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from `Send Reassignment Request`
  - Output to `Check Retry Deal Stage`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**  
  - Must explicitly configure 2-hour delay on rebuild
  - Requires persistent execution support
- **Sub-workflow reference:** None

---

## 2.10 Final Retry Decision and Manual Escalation

**Overview:**  
This block performs the second acceptance check. It either confirms the retry success or escalates for manual intervention, updates CRM accordingly, and records the outcome.

**Nodes Involved:**  
- Check Retry Deal Stage
- Consolidate Retry Status
- Retry Accepted?
- Update Deal — Confirmed (Retry)
- Send Customer — Confirmed (Retry)
- Log Success (Retry)
- Alert Slack — Manual Intervention
- Update Deal — Escalated
- Log Escalated

### Node Details

#### Check Retry Deal Stage
- **Type and technical role:** `n8n-nodes-base.freshworksCrm`; fetches current deal after retry window.
- **Configuration choices:**  
  - Resource: `deal`
  - Operation: `get`
  - Deal ID from `{{$json.deal_id}}`
- **Key expressions or variables used:**  
  - `={{ $json.deal_id }}`
- **Input and output connections:**  
  - Input from `Retry Wait — 2 Hours`
  - Output to `Consolidate Retry Status`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - Blank or invalid deal ID
- **Sub-workflow reference:** None

#### Consolidate Retry Status
- **Type and technical role:** `n8n-nodes-base.code`; interprets post-retry stage.
- **Configuration choices:**  
  - Same accepted-stage list as first SLA check
  - Sets:
    - `retry_deal_stage`
    - `retry_accepted`
    - `retry_check_at`
    - `sla_attempt = 2`
- **Key expressions or variables used:**  
  - `acceptedStages.some(s => stageName.includes(s))`
- **Input and output connections:**  
  - Input from `Check Retry Deal Stage`
  - Output to `Retry Accepted?`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - Same stage-name alignment issues as first SLA check
- **Sub-workflow reference:** None

#### Retry Accepted?
- **Type and technical role:** `n8n-nodes-base.if`; final branch between success and manual escalation.
- **Configuration choices:**  
  - Boolean equality on `retry_accepted`
- **Key expressions or variables used:**  
  - `={{ $json.retry_accepted }}`
- **Input and output connections:**  
  - Input from `Consolidate Retry Status`
  - True branch to `Update Deal — Confirmed (Retry)`
  - False branch to `Alert Slack — Manual Intervention`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**  
  - Depends fully on consistent Freshworks stage naming
- **Sub-workflow reference:** None

#### Update Deal — Confirmed (Retry)
- **Type and technical role:** `n8n-nodes-base.freshworksCrm`; updates deal for successful retry acceptance.
- **Configuration choices:**  
  - Resource: `deal`
  - Operation: `update`
  - `updateFields` empty
- **Key expressions or variables used:**  
  - `={{ $json.deal_id }}`
- **Input and output connections:**  
  - Input from true branch of `Retry Accepted?`
  - Output to `Send Customer — Confirmed (Retry)`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - No actual stage/status update unless `updateFields` are completed
- **Sub-workflow reference:** None

#### Send Customer — Confirmed (Retry)
- **Type and technical role:** `n8n-nodes-base.httpRequest`; intended Postmark customer email after retry success.
- **Configuration choices:**  
  - POST to Postmark
  - Body enabled but empty
- **Key expressions or variables used:** None currently in body.
- **Input and output connections:**  
  - Input from `Update Deal — Confirmed (Retry)`
  - Output to `Log Success (Retry)`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**  
  - Incomplete configuration for actual email delivery
- **Sub-workflow reference:** None

#### Log Success (Retry)
- **Type and technical role:** `n8n-nodes-base.googleSheets`; logs second-attempt success.
- **Configuration choices:**  
  - Status: `confirmed_retry`
  - SLA Attempt: `2`
  - Uses `Consolidate Retry Status` expressions
- **Key expressions or variables used:**  
  - `$('Consolidate Retry Status').first().json...`
  - `={{ DateTime.now().toISO() }}`
- **Input and output connections:**  
  - Input from `Send Customer — Confirmed (Retry)`
  - No downstream node
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**  
  - Same spreadsheet concerns as other logging nodes
- **Sub-workflow reference:** None

#### Alert Slack — Manual Intervention
- **Type and technical role:** `n8n-nodes-base.slack`; critical escalation for human handling.
- **Configuration choices:**  
  - Sends a detailed alert including:
    - both SLA failures
    - customer and service details
    - requested date
    - special notes
    - failed supplier 1
    - failed supplier 2
    - Freshworks deal ID
- **Key expressions or variables used:**  
  - `{{ $json.order_ref }}`
  - `{{ $json.customer_name }}`
  - `{{ $json.customer_email }}`
  - `{{ $json.customer_phone }}`
  - `{{ $json.service_type }}`
  - `{{ $json.skip_size || 'TBC' }}`
  - `{{ $json.collection_address }}`
  - `{{ $json.postcode }}`
  - `{{ $json.priority | upper }}`
  - `{{ $json.payment_amount_gbp }}`
  - `{{ $json.extracted.requested_date || 'ASAP' }}`
  - `{{ $json.special_instructions || 'None' }}`
  - `{{ $json.supplier_name }}`
  - `{{ $json.next_supplier_name }}`
  - `{{ $json.deal_id }}`
- **Input and output connections:**  
  - Input from false branch of `Retry Accepted?`
  - Output to `Update Deal — Escalated`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**  
  - Slack auth/channel issues
  - If `extracted` is undefined, expression fallback partly mitigates this
- **Sub-workflow reference:** None

#### Update Deal — Escalated
- **Type and technical role:** `n8n-nodes-base.freshworksCrm`; marks the deal escalated.
- **Configuration choices:**  
  - Resource: `deal`
  - Operation: `update`
  - `updateFields` empty
- **Key expressions or variables used:**  
  - `={{ $json.deal_id }}`
- **Input and output connections:**  
  - Input from `Alert Slack — Manual Intervention`
  - Output to `Log Escalated`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - As configured, no actual escalation field/stage is written unless completed
- **Sub-workflow reference:** None

#### Log Escalated
- **Type and technical role:** `n8n-nodes-base.googleSheets`; logs failed automated fulfillment requiring manual handling.
- **Configuration choices:**  
  - Status: `escalated_manual`
  - SLA Attempt: `2`
  - Logs both supplier names in one field
- **Key expressions or variables used:**  
  - `={{ $json.supplier_name }} / {{ $json.next_supplier_name }}`
  - `={{ DateTime.now().toISO() }}`
- **Input and output connections:**  
  - Input from `Update Deal — Escalated`
  - No downstream node
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**  
  - Spreadsheet ID/tab mismatch
- **Sub-workflow reference:** None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Visual documentation |  |  | ## Service Order Fulfillment & SLA Escalation Engine<br>Version 1.0.0 — Operations<br><br>Production-grade order fulfillment orchestrator for service brokerages (skip hire, field services, logistics). Verifies Stripe payment, extracts structured order details with Claude AI, creates CRM records in Freshworks, sends transactional emails via Postmark, assigns a supplier, enforces a 4-hour SLA, auto-escalates missed SLAs to Slack and reassigns to the next available supplier, then logs every outcome to Google Sheets.<br><br>Flow: Webhook => Validate Order => Verify Stripe Payment => Payment OK? => Extract with AI => Upsert Customer (Freshworks) => Create Deal => Send Confirmation (Postmark) => Find Supplier => Send Assignment => SLA Wait 4h => Check Deal Stage => Accepted? => Confirm/Escalate => SLA Retry 2h => Final Check => Log Outcome |
| Prerequisites | Sticky Note | Visual documentation |  |  | ## Prerequisites<br>- Stripe account with secret key (for payment verification)<br>- Freshworks CRM account — API key + domain (subdomain.freshsales.io)<br>- Postmark account — Server API Token (transactional email stream)<br>- Slack workspace — Bot Token + #ops-escalations channel<br>- Google Sheets — OAuth2 credential + pre-created fulfillment log sheet<br>- Anthropic API key (Claude for order extraction)<br><br>Freshworks setup: Go to Admin Settings → API Settings → copy your API key. Suppliers must be CRM contacts tagged 'supplier-available'. Deals need stages: New, Supplier Assigned, Accepted, Escalated, Completed. |
| Setup Required | Sticky Note | Visual documentation |  |  | ## Setup Required<br>1. Order Webhook — copy webhook URL into your website/checkout backend<br>2. Verify Stripe Payment — add Stripe Secret Key to the HTTP Request header<br>3. Extract Order Details — connect Anthropic API credential<br>4. Upsert Customer Contact — set FRESHWORKS_DOMAIN + FRESHWORKS_API_KEY<br>5. Create Service Deal — connect Freshworks CRM credential, update stageId for 'New'<br>6. Send Order Confirmation — add Postmark Server Token, update From address<br>7. Find Available Supplier — update tag filter to match your supplier tag in Freshworks<br>8. Send Supplier Assignment — update From address and email template<br>9. SLA Wait nodes — adjust 4h/2h windows to your SLA policy<br>10. Alert Slack — connect Slack credential, confirm channel name #ops-escalations<br>11. Log Outcome — connect Google Sheets, replace YOUR_FULFILLMENT_SHEET_ID |
| How It Works | Sticky Note | Visual documentation |  |  | ## How It Works<br>1. Webhook receives new order from checkout with payment_intent_id and customer details<br>2. Validate Code checks required fields and normalises data (email lowercase, postcode uppercase)<br>3. Stripe HTTP verifies payment_intent status = 'succeeded' before proceeding<br>4. Claude AI extracts waste type, skip size, collection address, priority from order notes<br>5. Customer contact upserted in Freshworks CRM via REST API<br>6. Service deal created in Freshworks with order value and all custom fields<br>7. Postmark sends branded order confirmation to customer<br>8. Freshworks searched for available supplier contacts (tagged supplier-available)<br>9. Postmark sends supplier assignment request with job details<br>10. Wait node holds execution for 4 hours (SLA window)<br>11. Deal stage checked — if Accepted, customer notified and success logged<br>12. If not: Slack alert fired, next supplier found, reassigned, 2h retry SLA starts<br>13. Retry check: if accepted → confirm; if not → Slack urgent alert + Escalated deal stage<br>14. All outcomes (success / retry success / escalated) logged to Google Sheets |
| New Order Webhook | Webhook | Receives incoming order POST |  | Validate & Parse Order |  |
| Validate note | Sticky Note | Visual documentation |  |  | Validates required fields, normalises email/postcode, assigns order reference |
| Validate & Parse Order | Code | Validates and normalizes order payload | New Order Webhook | Verify Stripe Payment | Validates required fields, normalises email/postcode, assigns order reference |
| Stripe note | Sticky Note | Visual documentation |  |  | Verifies payment_intent status = succeeded via Stripe API before processing |
| Verify Stripe Payment | HTTP Request | Calls Stripe Payment Intents API | Validate & Parse Order | Consolidate Payment Result | Verifies payment_intent status = succeeded via Stripe API before processing |
| Consolidate Payment Result | Code | Merges payment response with order context | Verify Stripe Payment | Payment Succeeded? |  |
| Payment check note | Sticky Note | Visual documentation |  |  | Routes to failure path if payment not succeeded — stops order processing immediately |
| Payment Succeeded? | IF | Branches on payment validity | Consolidate Payment Result | Extract Order Details; Send Payment Failed Notice | Routes to failure path if payment not succeeded — stops order processing immediately |
| Send Payment Failed Notice | HTTP Request | Intended Postmark payment-failure email | Payment Succeeded? | Log Failed Payment |  |
| Log Failed Payment | Google Sheets | Logs payment failure outcome | Send Payment Failed Notice |  |  |
| Extract note | Sticky Note | Visual documentation |  |  | Claude AI parses free-text order notes into a validated structured schema |
| Extract Order Details | LangChain Agent | Extracts structured order data with Claude | Payment Succeeded?; Claude — Extract; Order Details Schema | Upsert Customer Contact | Claude AI parses free-text order notes into a validated structured schema |
| Claude — Extract | Anthropic Chat Model | Provides Claude model for extraction |  | Extract Order Details |  |
| Order Details Schema | Structured Output Parser | Enforces JSON schema on AI output |  | Extract Order Details |  |
| Upsert note | Sticky Note | Visual documentation |  |  | Creates or updates customer contact in Freshworks CRM via upsert REST endpoint |
| Upsert Customer Contact | HTTP Request | Upserts customer contact in Freshworks | Extract Order Details | Consolidate Contact | Creates or updates customer contact in Freshworks CRM via upsert REST endpoint |
| Consolidate Contact | Code | Merges order, AI, payment, and contact data | Upsert Customer Contact | Create Service Deal |  |
| Deal note | Sticky Note | Visual documentation |  |  | Creates service deal in Freshworks CRM linked to the customer contact |
| Create Service Deal | Freshworks CRM | Creates service deal | Consolidate Contact | Consolidate Deal | Creates service deal in Freshworks CRM linked to the customer contact |
| Consolidate Deal | Code | Captures deal ID/name for downstream use | Create Service Deal | Send Order Confirmation |  |
| Confirmation note | Sticky Note | Visual documentation |  |  | Sends branded order confirmation email to customer via Postmark |
| Send Order Confirmation | HTTP Request | Intended customer confirmation email | Consolidate Deal | Find Available Supplier | Sends branded order confirmation email to customer via Postmark |
| Find Supplier note | Sticky Note | Visual documentation |  |  | Queries Freshworks for contacts tagged 'supplier-available' — picks the first match |
| Find Available Supplier | Freshworks CRM | Retrieves supplier candidates | Send Order Confirmation | Consolidate Supplier | Queries Freshworks for contacts tagged 'supplier-available' — picks the first match |
| Consolidate Supplier | Code | Chooses primary supplier | Find Available Supplier | Send Supplier Assignment |  |
| Supplier Request note | Sticky Note | Visual documentation |  |  | Sends job assignment request to supplier with full order details and acceptance deadline |
| Send Supplier Assignment | HTTP Request | Intended supplier assignment email | Consolidate Supplier | SLA Wait — 4 Hours | Sends job assignment request to supplier with full order details and acceptance deadline |
| SLA Wait note | Sticky Note | Visual documentation |  |  | Holds execution for 4 hours — the primary SLA window for supplier acceptance |
| SLA Wait — 4 Hours | Wait | Delays execution for first SLA window | Send Supplier Assignment | Check Deal Stage | Holds execution for 4 hours — the primary SLA window for supplier acceptance |
| Status check note | Sticky Note | Visual documentation |  |  | Polls Freshworks deal stage to determine if supplier has accepted the job |
| Check Deal Stage | Freshworks CRM | Retrieves deal stage after SLA wait | SLA Wait — 4 Hours | Consolidate SLA Status | Polls Freshworks deal stage to determine if supplier has accepted the job |
| Consolidate SLA Status | Code | Evaluates supplier acceptance from stage | Check Deal Stage | Supplier Accepted SLA? |  |
| Accepted check note | Sticky Note | Visual documentation |  |  | Routes TRUE if supplier accepted within SLA, FALSE triggers escalation path |
| Supplier Accepted SLA? | IF | Branches between success and escalation | Consolidate SLA Status | Update Deal — Confirmed; Alert Slack — SLA Missed | Routes TRUE if supplier accepted within SLA, FALSE triggers escalation path |
| Update Deal — Confirmed | Freshworks CRM | Intended deal update on first-attempt acceptance | Supplier Accepted SLA? | Send Customer — Job Confirmed |  |
| Send Customer — Job Confirmed | HTTP Request | Intended customer confirmation after acceptance | Update Deal — Confirmed | Log Success |  |
| Log Success | Google Sheets | Logs first-attempt success | Send Customer — Job Confirmed |  |  |
| Slack SLA note | Sticky Note | Visual documentation |  |  | Fires Slack alert to ops team with full order context when SLA is missed |
| Alert Slack — SLA Missed | Slack | Sends SLA miss alert to ops | Supplier Accepted SLA? | Find Next Supplier | Fires Slack alert to ops team with full order context when SLA is missed |
| Find Next Supplier | Freshworks CRM | Retrieves alternate supplier candidates | Alert Slack — SLA Missed | Consolidate Next Supplier |  |
| Consolidate Next Supplier | Code | Chooses next supplier excluding first one | Find Next Supplier | Send Reassignment Request |  |
| Reassign note | Sticky Note | Visual documentation |  |  | Sends reassignment request to next available supplier with escalation context |
| Send Reassignment Request | HTTP Request | Intended reassignment email | Consolidate Next Supplier | Retry Wait — 2 Hours | Sends reassignment request to next available supplier with escalation context |
| Retry wait note | Sticky Note | Visual documentation |  |  | Holds execution for 2 hours — the retry SLA window after reassignment |
| Retry Wait — 2 Hours | Wait | Delays execution for retry SLA window | Send Reassignment Request | Check Retry Deal Stage | Holds execution for 2 hours — the retry SLA window after reassignment |
| Check Retry Deal Stage | Freshworks CRM | Retrieves deal state after retry SLA | Retry Wait — 2 Hours | Consolidate Retry Status |  |
| Consolidate Retry Status | Code | Evaluates acceptance after retry | Check Retry Deal Stage | Retry Accepted? |  |
| Retry accepted note | Sticky Note | Visual documentation |  |  | Final decision: retry accepted triggers confirmation; failure triggers manual intervention alert |
| Retry Accepted? | IF | Branches retry success vs critical escalation | Consolidate Retry Status | Update Deal — Confirmed (Retry); Alert Slack — Manual Intervention | Final decision: retry accepted triggers confirmation; failure triggers manual intervention alert |
| Update Deal — Confirmed (Retry) | Freshworks CRM | Intended deal update after retry acceptance | Retry Accepted? | Send Customer — Confirmed (Retry) |  |
| Send Customer — Confirmed (Retry) | HTTP Request | Intended customer confirmation after retry success | Update Deal — Confirmed (Retry) | Log Success (Retry) |  |
| Log Success (Retry) | Google Sheets | Logs successful retry outcome | Send Customer — Confirmed (Retry) |  |  |
| Manual alert note | Sticky Note | Visual documentation |  |  | Both SLA attempts failed — urgent Slack alert for immediate manual intervention |
| Alert Slack — Manual Intervention | Slack | Sends critical manual intervention alert | Retry Accepted? | Update Deal — Escalated | Both SLA attempts failed — urgent Slack alert for immediate manual intervention |
| Update Deal — Escalated | Freshworks CRM | Intended deal escalation update | Alert Slack — Manual Intervention | Log Escalated |  |
| Log Escalated | Google Sheets | Logs manual escalation outcome | Update Deal — Escalated |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Automate service order fulfillment with Claude, Freshworks CRM and SLA escalation`.

2. **Add a Webhook node** named `New Order Webhook`.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `new-order`
   - Response mode: preferably configure a proper response strategy; if you keep `responseNode`, also add a response node in your own implementation.
   - Expected incoming fields:
     - `payment_intent_id`
     - `customer_email`
     - `customer_name`
     - `customer_phone`
     - `order_notes`
     - `service_type`
     - `collection_address`
     - `postcode`
     - `requested_date`
     - `order_value_gbp`
     - optional `order_ref`

3. **Add a Code node** named `Validate & Parse Order` and connect `New Order Webhook -> Validate & Parse Order`.
   - Paste logic equivalent to:
     - read request body
     - require `payment_intent_id`, `customer_email`, `order_notes`
     - lowercase email
     - validate email contains `@`
     - uppercase postcode
     - generate `order_ref` if missing
     - parse `order_value_gbp` into number
     - add `received_at`
   - Keep output as a single normalized item.

4. **Add an HTTP Request node** named `Verify Stripe Payment` and connect `Validate & Parse Order -> Verify Stripe Payment`.
   - Method: `GET`
   - URL: `https://api.stripe.com/v1/payment_intents/{{ $json.payment_intent_id }}`
   - Headers:
     - `Authorization: Bearer <YOUR_STRIPE_SECRET_KEY>`
     - `Stripe-Version: 2024-11-20.acacia`
   - Options:
     - enable retries: 3
     - wait between retries: 2000 ms
     - set response handling to tolerate non-2xx if you want to replicate the current design
   - Important: replace the placeholder token.

5. **Add a Code node** named `Consolidate Payment Result` and connect `Verify Stripe Payment -> Consolidate Payment Result`.
   - Merge:
     - original order from `Validate & Parse Order`
     - Stripe response from current input
   - Derive:
     - `payment_status`
     - `payment_valid = status === 'succeeded'`
     - `payment_amount_gbp = amount / 100`
     - `payment_currency`
     - `stripe_customer_id`
     - `payment_error`

6. **Add an IF node** named `Payment Succeeded?` and connect `Consolidate Payment Result -> Payment Succeeded?`.
   - Condition:
     - boolean equals true
     - left value: `{{ $json.payment_valid }}`

7. **Build the payment failure branch** from the false output.
   - Add `HTTP Request` node named `Send Payment Failed Notice`
   - Connect `Payment Succeeded? (false) -> Send Payment Failed Notice`
   - Configure for Postmark:
     - Method: `POST`
     - URL: `https://api.postmarkapp.com/email`
     - Headers:
       - `X-Postmark-Server-Token: <YOUR_POSTMARK_SERVER_TOKEN>`
       - `Accept: application/json`
       - `Content-Type: application/json`
     - Body should include at minimum:
       - `From`
       - `To`
       - `Subject`
       - `HtmlBody` or `TextBody`
   - Then add `Google Sheets` node named `Log Failed Payment`
   - Connect `Send Payment Failed Notice -> Log Failed Payment`
   - Configure:
     - Credential: Google Sheets OAuth2
     - Spreadsheet ID: your fulfillment log sheet
     - Sheet name: `Order Log`
     - Operation: append or append/update
     - Columns:
       - Reason
       - Status = `payment_failed`
       - Customer
       - Order Ref
       - Timestamp
       - Order Value

8. **Build the AI extraction branch** from the true output of `Payment Succeeded?`.
   - Add `LangChain Agent` node named `Extract Order Details`
   - Connect `Payment Succeeded? (true) -> Extract Order Details`
   - Set prompt to include:
     - order ref
     - service type
     - address
     - postcode
     - requested date
     - notes
     - estimated value
   - Add system instruction describing precise service-order extraction rules.

9. **Add the Anthropic model node** named `Claude — Extract`.
   - Type: Anthropic chat model
   - Model: `claude-3-5-sonnet-20241022`
   - Temperature: `0`
   - Connect it to `Extract Order Details` through the AI model port.
   - Configure Anthropic credentials with your API key.

10. **Add a Structured Output Parser node** named `Order Details Schema`.
    - Connect it to `Extract Order Details` through the output parser port.
    - Use a schema with fields:
      - `service_type`
      - `collection_address`
      - `postcode`
      - `waste_type`
      - `skip_size`
      - `requested_date`
      - `customer_name`
      - `customer_email`
      - `customer_phone`
      - `special_instructions`
      - `access_restrictions`
      - `estimated_value_gbp`
      - `priority`
    - Include enums for:
      - `waste_type`: `general`, `construction`, `garden`, `commercial`, `mixed`, `hazardous`, `other`
      - `priority`: `standard`, `urgent`, `emergency`

11. **Add an HTTP Request node** named `Upsert Customer Contact` and connect `Extract Order Details -> Upsert Customer Contact`.
    - Method: `POST`
    - URL: `https://<YOUR_FRESHSALES_DOMAIN>.freshsales.io/api/upsert/contacts`
    - Headers:
      - `Authorization: Token token=<YOUR_FRESHWORKS_API_KEY>`
      - `Content-Type: application/json`
    - Body should include actual contact payload, for example:
      - name
      - email
      - mobile/phone
      - address fields
      - custom attributes if needed
    - This body is missing in the source workflow and must be added for real use.

12. **Add a Code node** named `Consolidate Contact` and connect `Upsert Customer Contact -> Consolidate Contact`.
    - Merge:
      - original normalized order
      - payment context
      - AI output
      - Freshworks contact response
    - Produce:
      - `contact_id`
      - customer fields
      - extracted operational fields
      - payment fields

13. **Add a Freshworks CRM node** named `Create Service Deal` and connect `Consolidate Contact -> Create Service Deal`.
    - Resource: `Deal`
    - Operation: `Create`
    - Credential: Freshworks CRM
    - Name expression: use service type, postcode, and order ref
    - Set at minimum:
      - stage/stageId for `New`
      - value/amount
      - contact association
      - custom fields such as waste type, skip size, priority, notes
    - The source workflow only sets probability, so you must finish this configuration.

14. **Add a Code node** named `Consolidate Deal` and connect `Create Service Deal -> Consolidate Deal`.
    - Extract `deal_id` and `deal_name`
    - Merge with prior context

15. **Add an HTTP Request node** named `Send Order Confirmation` and connect `Consolidate Deal -> Send Order Confirmation`.
    - Configure Postmark as in step 7
    - Body should email the customer:
      - order reference
      - service summary
      - estimated date
      - contact/support instructions

16. **Add a Freshworks CRM node** named `Find Available Supplier` and connect `Send Order Confirmation -> Find Available Supplier`.
    - Resource: `Contact`
    - Operation: `Get Many` / `Get All`
    - Limit: `5`
    - Add filters so only supplier contacts tagged `supplier-available` are returned
    - Optionally filter by:
      - postcode/service area
      - service type capability
      - active status
    - This filtering is described in notes but not actually configured in the source workflow.

17. **Add a Code node** named `Consolidate Supplier` and connect `Find Available Supplier -> Consolidate Supplier`.
    - Choose the first valid supplier
    - Capture:
      - `supplier_id`
      - `supplier_email`
      - `supplier_name`
      - `supplier_phone`
      - `supplier_found`
      - `all_supplier_ids`
    - Add your own guard if no supplier is found.

18. **Add an HTTP Request node** named `Send Supplier Assignment` and connect `Consolidate Supplier -> Send Supplier Assignment`.
    - Configure Postmark
    - Email content should include:
      - order reference
      - service details
      - customer/location summary
      - requested date
      - acceptance deadline
      - deal link/reference
    - Ideally only send if `supplier_found` is true; add an IF node if needed.

19. **Add a Wait node** named `SLA Wait — 4 Hours` and connect `Send Supplier Assignment -> SLA Wait — 4 Hours`.
    - Configure explicit duration: `4 hours`
    - The source workflow node name implies this, but the actual duration is not set in the JSON.
    - Ensure your n8n instance supports long-running/persisted executions.

20. **Add a Freshworks CRM node** named `Check Deal Stage` and connect `SLA Wait — 4 Hours -> Check Deal Stage`.
    - Resource: `Deal`
    - Operation: `Get`
    - Deal ID: `{{ $json.deal_id }}`

21. **Add a Code node** named `Consolidate SLA Status` and connect `Check Deal Stage -> Consolidate SLA Status`.
    - Read the stage name
    - Normalize to lowercase
    - Mark accepted if stage contains one of:
      - accepted
      - confirmed
      - partner confirmed
      - supplier accepted
      - in progress
    - Set `sla_attempt = 1`

22. **Add an IF node** named `Supplier Accepted SLA?` and connect `Consolidate SLA Status -> Supplier Accepted SLA?`.
    - Condition: `{{ $json.supplier_accepted }}` equals `true`

23. **Build the first-success branch** from the true output.
    - Add `Freshworks CRM` node `Update Deal — Confirmed`
      - Operation: `Update`
      - Deal ID from context
      - Set stage to your confirmed/accepted stage
    - Add `HTTP Request` node `Send Customer — Job Confirmed`
      - Postmark email confirming supplier assignment
    - Add `Google Sheets` node `Log Success`
      - Log status `confirmed`
      - SLA attempt `1`
    - Connect:
      - `Supplier Accepted SLA? (true) -> Update Deal — Confirmed -> Send Customer — Job Confirmed -> Log Success`

24. **Build the SLA-missed branch** from the false output.
    - Add `Slack` node `Alert Slack — SLA Missed`
    - Connect `Supplier Accepted SLA? (false) -> Alert Slack — SLA Missed`
    - Configure Slack credentials
    - Send to your operations channel such as `#ops-escalations`
    - Include order/deal/supplier context in the message

25. **Add Freshworks CRM node** `Find Next Supplier` and connect `Alert Slack — SLA Missed -> Find Next Supplier`.
    - Resource: `Contact`
    - Operation: `Get All`
    - Limit: `10`
    - Again, apply supplier filters/tag rules

26. **Add Code node** `Consolidate Next Supplier` and connect `Find Next Supplier -> Consolidate Next Supplier`.
    - Exclude the original `supplier_id`
    - Select the first remaining supplier
    - Set:
      - `next_supplier_id`
      - `next_supplier_email`
      - `next_supplier_name`
      - `next_supplier_phone`
      - `next_supplier_found`

27. **Add HTTP Request node** `Send Reassignment Request` and connect `Consolidate Next Supplier -> Send Reassignment Request`.
    - Configure Postmark
    - Include escalation context and shorter response deadline

28. **Add Wait node** `Retry Wait — 2 Hours` and connect `Send Reassignment Request -> Retry Wait — 2 Hours`.
    - Configure explicit duration: `2 hours`
    - Again, the source JSON does not define this delay even though the node name does.

29. **Add Freshworks CRM node** `Check Retry Deal Stage` and connect `Retry Wait — 2 Hours -> Check Retry Deal Stage`.
    - Resource: `Deal`
    - Operation: `Get`
    - Deal ID: `{{ $json.deal_id }}`

30. **Add Code node** `Consolidate Retry Status` and connect `Check Retry Deal Stage -> Consolidate Retry Status`.
    - Reuse the same accepted-stage logic
    - Set `sla_attempt = 2`
    - Store `retry_accepted`

31. **Add IF node** `Retry Accepted?` and connect `Consolidate Retry Status -> Retry Accepted?`.
    - Condition: `{{ $json.retry_accepted }}` equals `true`

32. **Build the retry-success branch** from the true output.
    - Add `Freshworks CRM` node `Update Deal — Confirmed (Retry)`
      - Set stage to confirmed/accepted
    - Add `HTTP Request` node `Send Customer — Confirmed (Retry)`
      - Postmark customer email
    - Add `Google Sheets` node `Log Success (Retry)`
      - Status `confirmed_retry`
      - SLA attempt `2`
    - Connect in sequence.

33. **Build the manual-escalation branch** from the false output.
    - Add `Slack` node `Alert Slack — Manual Intervention`
      - Send critical message with both failed suppliers and full order context
    - Add `Freshworks CRM` node `Update Deal — Escalated`
      - Set stage to `Escalated`
    - Add `Google Sheets` node `Log Escalated`
      - Status `escalated_manual`
      - SLA attempt `2`
    - Connect:
      - `Retry Accepted? (false) -> Alert Slack — Manual Intervention -> Update Deal — Escalated -> Log Escalated`

34. **Configure credentials** for all integrations.
    - **Anthropic:** API key for Claude
    - **Freshworks CRM node:** native Freshworks credential
    - **Freshworks upsert HTTP node:** API key + domain in headers/URL
    - **Stripe:** secret key in Authorization header
    - **Postmark:** server token in headers
    - **Slack:** bot token / Slack app credential
    - **Google Sheets:** OAuth2 credential with access to the target spreadsheet

35. **Add missing production-grade safeguards** that are implied but not fully implemented in the JSON.
    - Add proper Postmark JSON bodies to all email nodes
    - Add real Freshworks request payload in `Upsert Customer Contact`
    - Add actual deal stage IDs/fields in all Freshworks update/create nodes
    - Add supplier filtering by tag
    - Add IF guards for no supplier found / no next supplier found
    - Add a response node if keeping webhook `responseMode=responseNode`

36. **Test the workflow** with sample payloads.
    - Case 1: valid paid order
    - Case 2: invalid payment intent
    - Case 3: no supplier found
    - Case 4: first supplier accepts
    - Case 5: first supplier misses SLA, second accepts
    - Case 6: both suppliers fail and manual escalation is triggered

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Service Order Fulfillment & SLA Escalation Engine, Version 1.0.0 — Operations | Workflow overview note |
| Intended use cases include service brokerages such as skip hire, field services, and logistics | Workflow overview note |
| Required integrations: Stripe, Freshworks CRM, Postmark, Slack, Google Sheets, Anthropic | Prerequisites note |
| Freshworks setup note: suppliers should be tagged `supplier-available`; deal stages should include `New`, `Supplier Assigned`, `Accepted`, `Escalated`, `Completed` | Freshworks configuration guidance |
| Setup checklist includes replacing placeholder values such as Stripe token, Postmark token, Freshworks domain/API key, and Google Sheet ID | Setup Required note |
| SLA policy in the notes is 4 hours for initial assignment and 2 hours for retry, but the Wait nodes are not actually configured with these durations in the JSON | Important implementation note |
| Several HTTP Request nodes for Postmark and Freshworks have body sending enabled but no actual body parameters configured | Important implementation note |
| Several Freshworks update nodes have empty `updateFields`, so they do not yet explicitly change deal stage/state | Important implementation note |
| The webhook uses `responseMode: responseNode`, but no response node is present in the workflow | Important implementation note |