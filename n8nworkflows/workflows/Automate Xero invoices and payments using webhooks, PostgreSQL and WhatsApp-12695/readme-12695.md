Automate Xero invoices and payments using webhooks, PostgreSQL and WhatsApp

https://n8nworkflows.xyz/workflows/automate-xero-invoices-and-payments-using-webhooks--postgresql-and-whatsapp-12695


# Automate Xero invoices and payments using webhooks, PostgreSQL and WhatsApp

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automate Xero invoices and payments using webhooks, PostgreSQL and WhatsApp

**Purpose / Use cases**
- Create **Xero sales invoices (ACCREC)** from an external system via **Webhook**.
- If created successfully, **log the invoice** to **PostgreSQL** (optional), create a **Google Calendar due-date event**, and send a **WhatsApp confirmation** via **Twilio**.
- Receive a **payment webhook**, update the invoice/logs, send a **WhatsApp payment confirmation**, delete the **Google Calendar event**, and respond to the webhook.

**Logical blocks (by dependencies)**

### 1.1 Invoice Creation Intake (Webhook → Xero)
Receives invoice payload and attempts invoice creation in Xero.

### 1.2 Invoice Post-Processing (IF success → DB log → Calendar → WhatsApp → Webhook response)
On successful Xero creation: optionally log to PostgreSQL, create a due-date calendar event, notify customer via WhatsApp, and return success response. Otherwise return error response.

### 1.3 Payment Intake & Settlement (Webhook → DB update → Xero update → WhatsApp → Calendar delete → Webhook response)
Receives payment payload, updates PostgreSQL (optional), updates the invoice (intended “paid”), notifies customer, removes calendar reminder, and responds success.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Invoice Creation Intake (Webhook → Xero)
**Overview:** Accepts a POST webhook payload describing an invoice and creates an **authorised** accounts receivable invoice in Xero.

**Nodes involved**
- Webhook Trigger - Invoice Created
- Xero - Create Invoice

#### Node: Webhook Trigger - Invoice Created
- **Type / role:** `Webhook` — entry point for invoice creation requests.
- **Configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `225363a4-5e45-46b5-aefe-8ca0a4fa27b2`
  - **Response mode:** “Respond with a Respond to Webhook node” (`responseNode`)
- **Key variables/fields expected:**
  - The workflow references `{{$json.body...}}` later, so the incoming request should contain a **JSON body** with fields like:
    - `dueDate`, `reference`, `invoiceNumber`
    - `totalAmount`
    - `lineItems[0].quantity`, `lineItems[0].unitAmount`, `lineItems[0].description`
- **Connections:**
  - Output → **Xero - Create Invoice**
- **Edge cases / failures:**
  - Non-JSON body or missing `body` object will break expressions downstream.
  - If no Respond to Webhook executes, the caller can hang until timeout; here both success/error paths end in Respond nodes, but only if the IF routes correctly.

#### Node: Xero - Create Invoice
- **Type / role:** `Xero` — creates an ACCREC invoice.
- **Configuration (interpreted):**
  - **Operation:** Create invoice (default for this node configuration)
  - **Invoice type:** `ACCREC`
  - **Organization ID:** `a5917db9-07f1-40dc-b5f6-c55b39730d0f`
  - **Contact:** fixed `contactId` = `Your_Contact_ID` (placeholder; not derived from webhook)
  - **Line item mapping (first line only):**
    - `description` ← `{{$json.body.lineItems[0].description}}`
    - `quantity` ← `{{$json.body.lineItems[0].quantity}}`
    - `unitAmount` ← `{{$json.body.lineItems[0].unitAmount}}`
    - `lineAmount` ← `{{$json.body.totalAmount}}`
    - `taxType` = `OUTPUT`
    - `taxAmount` ← `{{$json.body.totalAmount * 0.007}}` (0.7% computed tax)
    - `accountCode` = `200`
    - `itemCode` appears set to `"="` (likely unintended; may cause Xero validation issues)
  - **Additional fields:**
    - `status` = `AUTHORISED`
    - `dueDate` ← `{{$json.body.dueDate}}`
    - `reference` ← `{{$json.body.reference}}`
    - `invoiceNumber` ← `{{$json.body.invoiceNumber}}`
- **Credentials:** Xero OAuth2 credential (“Rill - Xero account”)
- **Connections:**
  - Output → **IF - Invoice Created Successfully**
- **Edge cases / failures:**
  - Invalid/expired OAuth token; missing required scopes; tenant/organisation mismatch.
  - Xero validation errors (invalid account code, invalid tax type, invalid dates).
  - If `lineItems[0]` is missing, expressions fail.
  - If the node returns data in a structure different than expected by the IF (e.g., `Status` vs `status`), success detection may fail.

---

### Block 1.2 — Invoice Post-Processing (IF → Postgres → Calendar → WhatsApp → Respond)
**Overview:** Verifies invoice creation succeeded, optionally logs to PostgreSQL, creates a calendar reminder for due date, sends WhatsApp confirmation, and responds to the webhook caller.

**Nodes involved**
- IF - Invoice Created Successfully
- Add Invoice Record (PostgreSQL)
- Google Calendar - Create Due Date Event
- WhatsApp - Send Invoice Confirmation (Twilio)
- Respond to Webhook - Success
- Respond to Webhook - Error

#### Node: IF - Invoice Created Successfully
- **Type / role:** `If` — routes execution based on Xero output.
- **Configuration (interpreted):**
  - Condition: `{{$json.Status}}` **equals** `"AUTHORISED"`
- **Connections:**
  - **True** → Add Invoice Record
  - **False** → Respond to Webhook - Error
- **Edge cases / failures:**
  - Field name mismatch: Xero often uses `Status` or `status` depending on node output mapping/version; if wrong, the IF may route to error even when successful.
  - If Xero returns an error object but still has no `Status`, IF falls to false branch; response uses `$json.error` which may not exist.

#### Node: Add Invoice Record
- **Type / role:** `Postgres` — inserts a log record into `public.invoice_logs`.
- **Configuration (interpreted):**
  - **Operation:** Insert (default; operation not explicitly set)
  - **Target:** schema `public`, table `invoice_logs`
  - **Mapped columns (key expressions):**
    - `Amount` ← `{{$json.AmountDue}}`
    - `Status` ← `{{$json.Contact.ContactStatus}}`
    - `Due Date` ← `{{$json.Date}}`
    - `timestamp` ← `{{$json.DateString}}`
    - `invoice_id` ← `{{$json.InvoiceID}}` (**required**)
    - `Client Name` ← `{{$json.Contact.Name}}`
- **Notes in node:** “Optional: Enable if using PostgreSQL for logging”
- **Connections:**
  - Output → Google Calendar - Create Due Date Event
- **Edge cases / failures:**
  - This node assumes the incoming JSON has Xero-like fields (`AmountDue`, `InvoiceID`, `Contact...`, `Date...`). If Xero node output is nested (common), these expressions may be wrong unless Xero node is configured to “simplify” output.
  - Table schema must match exactly (column names include spaces and capitalization).
  - If the table has constraints (e.g., unique invoice_id), inserts can fail.
  - Credential connectivity errors (Supabase Postgres often requires SSL, correct host/port).

#### Node: Google Calendar - Create Due Date Event
- **Type / role:** `Google Calendar` — creates a calendar event for the invoice due date.
- **Configuration (interpreted):**
  - **Calendar:** `user@example.com` (placeholder)
  - **Start:** `{{$json["Due Date"]}}T09:00:00`
  - **End:** `{{$json["Due Date"]}}T10:00:00`
  - **sendUpdates:** `all` (sends invitations/updates)
- **Credentials:** Google Calendar OAuth2
- **Connections:**
  - Output → WhatsApp - Send Invoice Confirmation
- **Edge cases / failures:**
  - If `Due Date` is not in `YYYY-MM-DD` format, event creation may fail.
  - Timezone handling: appending `T09:00:00` without timezone uses calendar/account defaults; may shift times.
  - If this node fails, WhatsApp + webhook success response will not run (caller may wait unless error-handled).

#### Node: WhatsApp - Send Invoice Confirmation
- **Type / role:** `Twilio` — sends a WhatsApp message confirming invoice creation.
- **Configuration (interpreted):**
  - `to`: `Recipient_number` (placeholder)
  - `from`: `Your_number` (placeholder; must be a Twilio WhatsApp-enabled sender)
  - `toWhatsapp`: true
  - Message template uses cross-node references:
    - `$('IF - Invoice Created Successfully').item.json.Contact.Name`
    - `...InvoiceNumber`, `...Total`, `...DueDateString`
- **Connections:**
  - Output → Respond to Webhook - Success
- **Edge cases / failures:**
  - Twilio sandbox/production requirements: recipient must have opted in (sandbox) or be permitted.
  - If IF node output item differs from current item index, `.item` reference can mismatch in multi-item runs.
  - If Xero output doesn’t include `DueDateString`/`Total`/`Contact.Name` at that path, message rendering fails.

#### Node: Respond to Webhook - Success
- **Type / role:** `Respond to Webhook` — returns success JSON to the invoice creation caller.
- **Configuration (interpreted):**
  - Respond with JSON:
    - `{"success": true, "message": "Invoice created and notifications sent", "invoiceId": $json.invoiceId}`
- **Connections:** terminal (no outputs)
- **Edge cases / failures:**
  - `$json.invoiceId` may not exist at this stage; likely the Xero invoice id is `InvoiceID` or similar. This can return `null` or error if strict evaluation applies.

#### Node: Respond to Webhook - Error
- **Type / role:** `Respond to Webhook` — returns error JSON to the invoice creation caller.
- **Configuration (interpreted):**
  - HTTP status code: **500**
  - Body:
    - `{"success": false, "message": "Invoice creation failed", "error": $json.error}`
- **Connections:** terminal
- **Edge cases / failures:**
  - If the preceding node doesn’t provide `$json.error`, the response may be uninformative.

---

### Block 1.3 — Payment Intake & Settlement (Webhook → Postgres → Xero → WhatsApp → Calendar → Respond)
**Overview:** Accepts a payment notification, updates the log (optional), updates the invoice in Xero (intended as “paid”), sends WhatsApp confirmation, deletes the due-date calendar event, and responds to the payment webhook.

**Nodes involved**
- Webhook - Payment Received
- PostgreSQL - Update Payment Status
- Xero - Update Invoice to Paid1
- WhatsApp - Send Invoice Confirmation1
- Google Calendar - Remove Due Date Event
- Respond to Webhook - Payment

#### Node: Webhook - Payment Received
- **Type / role:** `Webhook` — entry point for payment notifications.
- **Configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `1f435bd2-cbfe-4c4d-9cbd-cc3bf4f8e10a`
  - **Response mode:** `responseNode`
- **Expected payload fields (as used later):**
  - `body.invoiceId`
  - `body.paymentDate`
  - Potentially also: `clientName`, `paymentAmount`, `invoiceNumber`, `transactionId`, `calendarEventId` (used later but not consistently under `body`)
- **Connections:**
  - Output → PostgreSQL - Update Payment Status
- **Edge cases / failures:**
  - Field placement inconsistency: later nodes reference both `$json.body.*` and `$json.*` top-level fields; if webhook does not match, messages/deletes can fail.

#### Node: PostgreSQL - Update Payment Status
- **Type / role:** `Postgres` — updates `invoice_logs` with payment date for an invoice.
- **Configuration (interpreted):**
  - **Operation:** Update
  - **Matching columns:** `id` and `invoice_id`
  - **Values set:**
    - `invoice_id` ← `{{ $json.body.invoiceId }}` (note: missing leading `=`; see edge case)
    - `Payment Date` ← `{{$json.body.paymentDate}}`
  - `id` is set to `0` (likely placeholder; will prevent matching unless row id is 0)
- **Notes in node:** “Optional: Enable if using PostgreSQL for logging”
- **Connections:**
  - Output → Xero - Update Invoice to Paid1
- **Edge cases / failures:**
  - The expression for `invoice_id` is `{{ $json.body.invoiceId }}` without `=` prefix. In n8n, fields typically need `={{ ... }}` to evaluate; otherwise it may be treated as a literal string, depending on parameter type.
  - Matching on `id=0` likely updates nothing. Better to match only on `invoice_id`.
  - Column names with spaces must match DB schema exactly.

#### Node: Xero - Update Invoice to Paid1
- **Type / role:** `Xero` — updates an invoice by ID (intended to mark as paid).
- **Configuration (interpreted):**
  - **Operation:** Update
  - **Invoice ID:** `{{$json.body.invoiceId}}`
  - **Update fields:** empty (`updateFields: {}`), so it may perform a no-op update.
  - **Organization ID:** same as above.
- **Connections:**
  - Output → WhatsApp - Send Invoice Confirmation1
- **Edge cases / failures:**
  - In Xero, “paid” is typically achieved by creating a **Payment** against the invoice, not merely updating invoice fields. With empty update fields, this likely won’t mark it paid.
  - If webhook provides invoice number rather than Xero invoice GUID, update will fail.

#### Node: WhatsApp - Send Invoice Confirmation1
- **Type / role:** `Twilio` — sends a payment received WhatsApp message.
- **Configuration (interpreted):**
  - `to`, `from`: placeholders
  - Message uses top-level fields:
    - `{{$json.clientName}}`, `{{$json.paymentAmount}}`, `{{$json.invoiceNumber}}`, `{{$json.transactionId}}`
- **Connections:**
  - Output → Google Calendar - Remove Due Date Event
- **Edge cases / failures:**
  - If payment webhook nests these under `body`, message will render blanks.
  - Same Twilio WhatsApp constraints as above.

#### Node: Google Calendar - Remove Due Date Event
- **Type / role:** `Google Calendar` — deletes an event that represented the invoice due reminder.
- **Configuration (interpreted):**
  - **Operation:** Delete
  - **Calendar:** `user@example.com`
  - **Event ID:** `{{$json.calendarEventId}}`
- **Connections:**
  - Output → Respond to Webhook - Payment
- **Edge cases / failures:**
  - Requires the payment flow to know `calendarEventId`. The invoice creation flow does not store the created event id into PostgreSQL in this workflow, so the payment webhook must supply it, or deletion will fail.
  - If event already deleted or ID invalid, Google returns 404.

#### Node: Respond to Webhook - Payment
- **Type / role:** `Respond to Webhook` — returns success JSON to the payment caller.
- **Configuration (interpreted):**
  - `{"success": true, "message": "Payment processed successfully"}`
- **Connections:** terminal

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger - Invoice Created | Webhook | Entry point for invoice creation | — | Xero - Create Invoice | ## Xero Invoice and Payment Automation Using n8n, PostgreSQL, and WhatsApp<br>## How it works<br>- **Webhook Trigger nodes** call the data from the source<br>- **Xero nodes** are used to create and update invoices<br>- **If node** to verify if invoice is successfully created<br>- **PostgreSQL nodes serve as a database to log invoice records and update<br>- Google Calendar nodes** are to log and modify the calendar event<br>- **Twilio nodes** are to send WhatsApp notifications<br>- **Respond webhook nodes** to send webhook responses<br><br>## Setup<br>1. Go to [xero](https://www.xero.com/), create an App, get details and add to credentials, then match<br>2. Go to [Supabase](https://supabase.com/), create a table with the required details, connect and add to PostgreSQL credentials. P.S. This is not a mistake<br>3. Enable Google Calendar from [Google Cloud console](https://console.cloud.google.com/) and connect to credentials<br>4. Go to [Twilio](https://www.twilio.com/en-us) and connect to n8n credential. Get your WhatsApp number there, too<br>5. Match the webhook to the payment data source and match the data in the nodes |
| Xero - Create Invoice | Xero | Create ACCREC invoice in Xero | Webhook Trigger - Invoice Created | IF - Invoice Created Successfully | ## 1. Create invoice, send notification and log |
| IF - Invoice Created Successfully | If | Validate invoice creation succeeded | Xero - Create Invoice | Add Invoice Record; Respond to Webhook - Error | ## 1. Create invoice, send notification and log |
| Add Invoice Record | Postgres | Insert invoice record into `invoice_logs` (optional) | IF - Invoice Created Successfully (true) | Google Calendar - Create Due Date Event | ## 1. Create invoice, send notification and log |
| Google Calendar - Create Due Date Event | Google Calendar | Create due-date reminder event | Add Invoice Record | WhatsApp - Send Invoice Confirmation | ## 1. Create invoice, send notification and log |
| WhatsApp - Send Invoice Confirmation | Twilio | Send WhatsApp invoice created message | Google Calendar - Create Due Date Event | Respond to Webhook - Success | ## 1. Create invoice, send notification and log |
| Respond to Webhook - Success | Respond to Webhook | Return success response for invoice webhook | WhatsApp - Send Invoice Confirmation | — | ## 1. Create invoice, send notification and log |
| Respond to Webhook - Error | Respond to Webhook | Return error response for invoice webhook | IF - Invoice Created Successfully (false) | — | ## 1. Create invoice, send notification and log |
| Webhook - Payment Received | Webhook | Entry point for payment notification | — | PostgreSQL - Update Payment Status | ## 2. Check and update payments, send notification and cancel event |
| PostgreSQL - Update Payment Status | Postgres | Update payment date/status in DB (optional) | Webhook - Payment Received | Xero - Update Invoice to Paid1 | ## 2. Check and update payments, send notification and cancel event |
| Xero - Update Invoice to Paid1 | Xero | Update invoice after payment (intended “paid”) | PostgreSQL - Update Payment Status | WhatsApp - Send Invoice Confirmation1 | ## 2. Check and update payments, send notification and cancel event |
| WhatsApp - Send Invoice Confirmation1 | Twilio | Send WhatsApp payment received message | Xero - Update Invoice to Paid1 | Google Calendar - Remove Due Date Event | ## 2. Check and update payments, send notification and cancel event |
| Google Calendar - Remove Due Date Event | Google Calendar | Delete due-date reminder event | WhatsApp - Send Invoice Confirmation1 | Respond to Webhook - Payment | ## 2. Check and update payments, send notification and cancel event |
| Respond to Webhook - Payment | Respond to Webhook | Return success response for payment webhook | Google Calendar - Remove Due Date Event | — | ## 2. Check and update payments, send notification and cancel event |
| Sticky Note | Sticky Note | Comment block header | — | — | ## 1. Create invoice, send notification and log |
| Sticky Note1 | Sticky Note | Comment block header | — | — | ## 2. Check and update payments, send notification and cancel event |
| Sticky Note2 | Sticky Note | Global description/setup notes | — | — | ## Xero Invoice and Payment Automation Using n8n, PostgreSQL, and WhatsApp<br>(contains setup links) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **Xero OAuth2** credential in n8n:
      - Create a Xero App in Xero Developer portal, set redirect URL to your n8n callback, authorize, select the correct tenant.
   2. **PostgreSQL** credential (optional):
      - If using Supabase Postgres, use Supabase connection parameters (host, db, user, password, SSL settings as required).
   3. **Google Calendar OAuth2** credential:
      - Enable Google Calendar API in Google Cloud Console, configure OAuth consent + credentials, connect in n8n.
   4. **Twilio** credential:
      - Add Twilio Account SID/Auth Token; ensure WhatsApp sender is enabled (sandbox or approved sender).

2) **Invoice creation flow**
   1. Add node **Webhook** named **“Webhook Trigger - Invoice Created”**
      - Method: POST
      - Response Mode: **Using “Respond to Webhook” node**
      - Path: choose a unique path (can match the given UUID).
   2. Add node **Xero** named **“Xero - Create Invoice”**
      - Resource/operation: Create Invoice
      - Type: **ACCREC**
      - Organization/Tenant: set your org ID
      - Contact: set a valid `contactId` (or modify to map from webhook)
      - Line items: map from webhook body (quantity, unit amount, description, etc.)
      - Status: **AUTHORISED**
      - DueDate/Reference/InvoiceNumber: map from webhook body
      - Select Xero OAuth2 credential.
   3. Add node **If** named **“IF - Invoice Created Successfully”**
      - Condition: invoice status equals `AUTHORISED` (adjust field path to match actual Xero output).
   4. Add node **Postgres** named **“Add Invoice Record”** (optional)
      - Schema: `public`
      - Table: `invoice_logs`
      - Operation: Insert
      - Map columns you want to store (invoice ID, client name, due date, amount, timestamps).
      - Select Postgres credential.
   5. Add node **Google Calendar** named **“Google Calendar - Create Due Date Event”**
      - Operation: Create event
      - Calendar: choose target calendar
      - Start/End: build datetime from due date (ensure correct date format/timezone)
      - sendUpdates: `all` if you want notifications.
   6. Add node **Twilio** named **“WhatsApp - Send Invoice Confirmation”**
      - Enable WhatsApp sending (`toWhatsapp: true`)
      - Set `from` to your Twilio WhatsApp-enabled number
      - Set `to` to recipient number (or map from webhook/customer record)
      - Message: reference invoice fields from Xero output.
   7. Add node **Respond to Webhook** named **“Respond to Webhook - Success”**
      - Respond with JSON `{ success: true, message: "...", invoiceId: ... }`
   8. Add node **Respond to Webhook** named **“Respond to Webhook - Error”**
      - Response code: 500
      - JSON `{ success: false, message: "...", error: ... }`

   **Connect nodes**
   - Webhook Trigger - Invoice Created → Xero - Create Invoice → IF
   - IF (true) → Add Invoice Record → Google Calendar - Create Due Date Event → WhatsApp - Send Invoice Confirmation → Respond to Webhook - Success
   - IF (false) → Respond to Webhook - Error

3) **Payment processing flow**
   1. Add node **Webhook** named **“Webhook - Payment Received”**
      - POST, response via Respond node, unique path.
   2. Add node **Postgres** named **“PostgreSQL - Update Payment Status”** (optional)
      - Operation: Update
      - Match on `invoice_id` (recommended) rather than `id=0`
      - Set `Payment Date` from webhook payload
   3. Add node **Xero** named **“Xero - Update Invoice to Paid1”**
      - Operation: Update invoice by ID
      - Provide invoiceId from webhook payload
      - (Recommended change) use a Xero “Create Payment” operation if you truly need to mark it paid.
   4. Add node **Twilio** named **“WhatsApp - Send Invoice Confirmation1”**
      - Send payment received message; map client name, amount, invoice number, transaction id.
   5. Add node **Google Calendar** named **“Google Calendar - Remove Due Date Event”**
      - Operation: Delete event
      - Provide `eventId` (you must have stored it earlier or receive it in the payment webhook).
   6. Add node **Respond to Webhook** named **“Respond to Webhook - Payment”**
      - Respond with JSON success message.

   **Connect nodes**
   - Webhook - Payment Received → PostgreSQL - Update Payment Status → Xero - Update Invoice to Paid1 → WhatsApp - Send Invoice Confirmation1 → Google Calendar - Remove Due Date Event → Respond to Webhook - Payment

4) **Add sticky notes (optional documentation)**
   - Add one sticky note covering the invoice creation chain: “## 1. Create invoice, send notification and log”
   - Add one sticky note covering the payment chain: “## 2. Check and update payments, send notification and cancel event”
   - Add one sticky note with the global setup links/content.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Go to Xero, create an App, get details and add to credentials | https://www.xero.com/ |
| Go to Supabase, create a table, connect and add to PostgreSQL credentials. “P.S. This is not a mistake” | https://supabase.com/ |
| Enable Google Calendar API and connect to credentials | https://console.cloud.google.com/ |
| Connect Twilio credential; obtain WhatsApp number | https://www.twilio.com/en-us |
| Ensure webhook payload fields match the expressions used in nodes | Applies to both invoice and payment webhooks |

