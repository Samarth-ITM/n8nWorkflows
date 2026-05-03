Generate and track invoices with HubSpot, Gmail, Slack, Sheets, and Notion

https://n8nworkflows.xyz/workflows/generate-and-track-invoices-with-hubspot--gmail--slack--sheets--and-notion-14289


# Generate and track invoices with HubSpot, Gmail, Slack, Sheets, and Notion

# 1. Workflow Overview

This workflow automates invoice generation and payment follow-up after a HubSpot deal is marked as won. It listens for HubSpot deal property changes, filters for deals moved to `closedwon`, retrieves deal and contact details, builds an HTML invoice, converts it to PDF, logs the invoice in Google Sheets and Notion, emails the customer via Gmail, alerts the team in Slack, then monitors payment status with delayed checks and escalation steps.

Primary use cases:
- Automatically invoice customers when a sales deal closes
- Maintain invoice tracking across Sheets and Notion
- Notify internal teams when invoices are sent, paid, or overdue
- Automate first and second-stage follow-ups for unpaid invoices
- Alert operators when the workflow itself fails

## 1.1 Trigger & Deal Qualification
The workflow starts from a HubSpot trigger that fires on deal property changes. An IF node ensures only stage updates where the property becomes `closedwon` continue into invoice processing.

## 1.2 Deal and Contact Data Retrieval
Once a qualified deal is detected, the workflow fetches full deal details, retrieves associated contacts, extracts the first contact ID, and loads that contact’s details from HubSpot.

## 1.3 Invoice Data Assembly
Using deal and contact information, a Code node generates invoice metadata such as invoice number, issue date, due date, formatted amount, and a full styled HTML invoice document.

## 1.4 PDF Creation, Logging, and Sending
The HTML invoice is converted to PDF through html2pdf.app, then the workflow logs invoice details into Google Sheets, creates a record in Notion, emails the customer through Gmail, and posts a Slack success notification.

## 1.5 Payment Monitoring and Follow-up
After a 7-day wait, the workflow checks the HubSpot deal stage again. If payment is confirmed via a target paid stage (`closedwon_paid`), it posts a Slack confirmation. Otherwise, it sends a follow-up email, waits 5 more days, performs a final check, and either escalates overdue invoices or reports a late payment confirmation.

## 1.6 Error Handling
A separate error-trigger branch captures any workflow execution failure and posts a formatted Slack alert with execution details for manual intervention.

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger & Deal Qualification

### Overview
This block receives HubSpot deal update events and filters them so only deals that changed into the `closedwon` stage enter the invoicing pipeline. It prevents invoice generation on unrelated HubSpot property changes.

### Nodes Involved
- `🎯 HubSpot - Deal Trigger`
- `🔀 IF - Is Deal Closed Won?`

### Node Details

#### 1. `🎯 HubSpot - Deal Trigger`
- **Type and technical role:** HubSpot Trigger node; event-based workflow entry point.
- **Configuration choices:** Configured to listen to HubSpot event `deal.propertyChange`.
- **Key expressions or variables used:** Outputs HubSpot webhook payload fields such as `objectId`, `propertyName`, and `propertyValue`.
- **Input and output connections:** No input; outputs to `🔀 IF - Is Deal Closed Won?`.
- **Version-specific requirements:** Uses `typeVersion: 1`; requires a working HubSpot trigger setup in n8n and HubSpot app/webhook connectivity.
- **Edge cases or potential failure types:**
  - HubSpot app authorization issues
  - Trigger not firing if webhook/app not configured properly
  - Payload shape differences depending on HubSpot event configuration
  - Duplicate trigger events if HubSpot emits multiple property-change notifications
- **Sub-workflow reference:** None.

#### 2. `🔀 IF - Is Deal Closed Won?`
- **Type and technical role:** IF node; gatekeeper filter.
- **Configuration choices:** Two string conditions are defined:
  - `propertyName == dealstage`
  - `propertyValue == closedwon`
  This means only a deal stage change to `closedwon` passes.
- **Key expressions or variables used:**
  - `{{ $json.propertyName }}`
  - `{{ $json.propertyValue }}`
- **Input and output connections:** Input from `🎯 HubSpot - Deal Trigger`; true output goes to `🏷️ HTTP - Get Deal Details`. False branch is unused.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - HubSpot stage internal value may not actually be `closedwon` in a given portal
  - If stage labels differ from internal IDs, the condition may never match
  - If other property changes accompany stage changes, this logic is correct only if the specific event payload contains the expected fields
- **Sub-workflow reference:** None.

---

## Block 2 — Deal and Contact Data Retrieval

### Overview
This block enriches the trigger payload by retrieving deal details, associated contacts, and the target contact record used for billing. It also validates that at least one contact is associated with the deal.

### Nodes Involved
- `🏷️ HTTP - Get Deal Details`
- `🔗 HTTP - Get Deal Associations`
- `⚙️ Code - Extract Contact ID`
- `👤 HTTP - Get Contact Details`

### Node Details

#### 3. `🏷️ HTTP - Get Deal Details`
- **Type and technical role:** HTTP Request node; retrieves the full HubSpot deal object.
- **Configuration choices:** Sends a GET request to HubSpot CRM v3 object API using the trigger’s `objectId`. Requests these properties:
  - `dealname`
  - `amount`
  - `dealstage`
  - `closedate`
  - `hubspot_owner_id`
  - `description`
- **Key expressions or variables used:**
  - `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.objectId }}?...`
- **Input and output connections:** Input from `🔀 IF - Is Deal Closed Won?`; output to `🔗 HTTP - Get Deal Associations`.
- **Version-specific requirements:** `typeVersion: 4`; uses generic credential type with `httpHeaderAuth`.
- **Edge cases or potential failure types:**
  - Invalid or missing HubSpot auth header
  - `objectId` missing in payload
  - 404 if the deal no longer exists
  - Rate limiting from HubSpot
- **Sub-workflow reference:** None.

#### 4. `🔗 HTTP - Get Deal Associations`
- **Type and technical role:** HTTP Request node; retrieves deal-to-contact associations from HubSpot.
- **Configuration choices:** GET request to `/crm/v3/objects/deals/{{ $json.id }}/associations/contacts`.
- **Key expressions or variables used:**
  - Uses the deal ID from the previous node: `{{ $json.id }}`
- **Input and output connections:** Input from `🏷️ HTTP - Get Deal Details`; output to `⚙️ Code - Extract Contact ID`.
- **Version-specific requirements:** `typeVersion: 4`; uses generic HTTP header auth.
- **Edge cases or potential failure types:**
  - Missing auth credential
  - Deal with no associated contacts
  - HubSpot API schema changes or permission issues
- **Sub-workflow reference:** None.

#### 5. `⚙️ Code - Extract Contact ID`
- **Type and technical role:** Code node; validates associations and merges selected deal data into a compact structure.
- **Configuration choices:** JavaScript extracts the first associated contact from `results[0].id`. If none exist, it throws an explicit error. It also reaches back to `🏷️ HTTP - Get Deal Details` to pull deal fields.
- **Key expressions or variables used:**
  - `$input.first().json.results`
  - `$('🏷️ HTTP - Get Deal Details').first().json`
- **Returned fields:**
  - `contactId`
  - `dealId`
  - `dealName`
  - `amount`
  - `closeDate`
- **Defaults used:**
  - `dealName`: `'Professional Services'`
  - `amount`: `'0'`
  - `closeDate`: current ISO date if missing
- **Input and output connections:** Input from `🔗 HTTP - Get Deal Associations`; output to `👤 HTTP - Get Contact Details`.
- **Version-specific requirements:** `typeVersion: 2`; cross-node references require stable node names.
- **Edge cases or potential failure types:**
  - No associated contact causes intentional workflow error
  - If multiple contacts exist, only the first is used
  - Renaming upstream nodes without updating expressions breaks the code
  - Deal payload missing expected properties
- **Sub-workflow reference:** None.

#### 6. `👤 HTTP - Get Contact Details`
- **Type and technical role:** HTTP Request node; loads billing contact data from HubSpot.
- **Configuration choices:** GET request to contact endpoint with selected properties:
  - `firstname`
  - `lastname`
  - `email`
  - `company`
  - `phone`
  - `address`
- **Key expressions or variables used:**
  - `https://api.hubapi.com/crm/v3/objects/contacts/{{ $json.contactId }}?...`
- **Input and output connections:** Input from `⚙️ Code - Extract Contact ID`; output to `📄 Code - Build Invoice + HTML`.
- **Version-specific requirements:** `typeVersion: 4`.
- **Edge cases or potential failure types:**
  - In the JSON provided, credentials are not explicitly attached, unlike some other HTTP nodes; if not inherited or configured manually, this node can fail at runtime
  - Missing contact email or company fields
  - Deleted or inaccessible contact
- **Sub-workflow reference:** None.

---

## Block 3 — Invoice Data Assembly

### Overview
This block builds all invoice business data and produces a complete HTML invoice template. It centralizes invoice numbering, due-date calculation, formatting, and invoice branding.

### Nodes Involved
- `📄 Code - Build Invoice + HTML`

### Node Details

#### 7. `📄 Code - Build Invoice + HTML`
- **Type and technical role:** Code node; transforms contact and deal data into invoice metadata plus styled HTML content.
- **Configuration choices:** The script:
  - Reads contact data from the current input
  - Reads prior deal data from `⚙️ Code - Extract Contact ID`
  - Generates invoice number in the form `INV-YYYY-####`
  - Sets `issue_date` to today
  - Sets `due_date` to 30 days from now
  - Parses and formats amount
  - Builds a full HTML document with CSS styling
- **Key expressions or variables used:**
  - `$input.first().json.properties`
  - `$input.first().json.id`
  - `$('⚙️ Code - Extract Contact ID').first().json`
- **Returned fields:**
  - `invoice_number`
  - `issue_date`
  - `due_date`
  - `amount`
  - `formatted_amount`
  - `deal_name`
  - `contact_name`
  - `contact_email`
  - `company`
  - `phone`
  - `deal_id`
  - `contact_id`
  - `html_content`
- **Branding placeholders that must be customized:**
  - `YOUR COMPANY NAME`
  - `123 Business St, City, State 12345`
  - `user@example.com`
  - `+1234567890`
  - bank details in payment instructions
- **Input and output connections:** Input from `👤 HTTP - Get Contact Details`; output to `🖨️ HTTP - Generate PDF`.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Random invoice numbers may collide over time because no uniqueness check is performed
  - Missing contact email still allows invoice creation but may break later Gmail sending
  - `parseFloat` on malformed `amount` may yield `NaN`
  - HTML content may break PDF rendering if special characters are not escaped in source data
- **Sub-workflow reference:** None.

---

## Block 4 — PDF Creation, Logging, and Sending

### Overview
This block converts the generated HTML into a PDF, stores invoice metadata in operational tools, sends the invoice to the client, and notifies the internal team.

### Nodes Involved
- `🖨️ HTTP - Generate PDF`
- `📊 Google Sheets - Log Invoice`
- `📓 Notion - Create Invoice Record`
- `📧 Gmail - Send Invoice Email`
- `💬 Slack - Invoice Sent Alert`

### Node Details

#### 8. `🖨️ HTTP - Generate PDF`
- **Type and technical role:** HTTP Request node; external document rendering service call.
- **Configuration choices:** POST to `https://api.html2pdf.app/v1/generate` with body fields:
  - `html`
  - `apiKey`
  - `media_type = print`
  - `format = A4`
  - `landscape = false`
  Response is configured as a file stored in binary property `invoice_pdf`.
- **Key expressions or variables used:**
  - `{{ $json.html_content }}`
- **Input and output connections:** Input from `📄 Code - Build Invoice + HTML`; output to `📊 Google Sheets - Log Invoice`.
- **Version-specific requirements:** `typeVersion: 4`; response format must be set to file.
- **Edge cases or potential failure types:**
  - Placeholder API key `REPLACE_HTML2PDF_API_KEY` must be replaced
  - Service downtime or HTML rendering failure
  - Large or malformed HTML may cause timeout
  - Binary output is generated but not actually attached downstream in the Gmail node as configured here
- **Sub-workflow reference:** None.

#### 9. `📊 Google Sheets - Log Invoice`
- **Type and technical role:** Google Sheets node; appends invoice metadata to a spreadsheet.
- **Configuration choices:** Append operation into `Sheet1` of a target spreadsheet. Columns mapped explicitly:
  - Email
  - Amount
  - Status
  - Company
  - Due Date
  - Deal Name
  - Issue Date
  - Client Name
  - Invoice Number
  - HubSpot Deal ID
- **Key expressions or variables used:** Pulls all values from `📄 Code - Build Invoice + HTML`.
- **Input and output connections:** Input from `🖨️ HTTP - Generate PDF`; output to `📓 Notion - Create Invoice Record`.
- **Version-specific requirements:** `typeVersion: 4`; spreadsheet ID `REPLACE_GOOGLE_SHEET_ID` must be replaced.
- **Edge cases or potential failure types:**
  - Invalid Google credentials
  - Missing spreadsheet access permissions
  - Sheet name mismatch
  - Column headers not matching expected names
- **Sub-workflow reference:** None.

#### 10. `📓 Notion - Create Invoice Record`
- **Type and technical role:** Notion node; creates an invoice record/page.
- **Configuration choices:** Title is set to:
  - `invoice_number — contact_name`
  The `pageId` is currently blank despite being in URL mode.
- **Key expressions or variables used:**
  - `{{ $('📄 Code - Build Invoice + HTML').item.json.invoice_number }}`
  - `{{ $('📄 Code - Build Invoice + HTML').item.json.contact_name }}`
- **Input and output connections:** Input from `📊 Google Sheets - Log Invoice`; output to `📧 Gmail - Send Invoice Email`.
- **Version-specific requirements:** `typeVersion: 2`; requires valid Notion credentials and a valid parent page/database target.
- **Edge cases or potential failure types:**
  - As provided, `pageId` is empty, so this node is incomplete and must be configured before use
  - If intended to create a database row rather than a page, the selected resource/operation may need adjustment
  - Missing permissions in Notion integration
- **Sub-workflow reference:** None.

#### 11. `📧 Gmail - Send Invoice Email`
- **Type and technical role:** Gmail node; sends the invoice email to the client.
- **Configuration choices:** Sends HTML email to the billing contact with subject and body built from invoice data.
- **Key expressions or variables used:**
  - Recipient: `{{ $('📄 Code - Build Invoice + HTML').item.json.contact_email }}`
  - Subject and message interpolate invoice number, amount, due date, and contact name
- **Input and output connections:** Input from `📓 Notion - Create Invoice Record`; output to `💬 Slack - Invoice Sent Alert`.
- **Version-specific requirements:** `typeVersion: 2`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Missing or invalid contact email
  - Gmail quota limits or OAuth consent issues
  - The message says the invoice PDF is attached, but no attachment configuration is present in the node as shown
  - HTML rendering in email clients may vary
- **Sub-workflow reference:** None.

#### 12. `💬 Slack - Invoice Sent Alert`
- **Type and technical role:** Slack node; internal notification after invoice email send.
- **Configuration choices:** Sends a Markdown-enabled message summarizing invoice, client, amount, deal, due date, and a note that auto follow-up will happen in 7 days.
- **Key expressions or variables used:** References fields from `📄 Code - Build Invoice + HTML`.
- **Input and output connections:** Input from `📧 Gmail - Send Invoice Email`; output to `⏳ Wait - 7 Day Payment Window`.
- **Version-specific requirements:** `typeVersion: 2`; Slack OAuth2 authentication required.
- **Edge cases or potential failure types:**
  - Invalid Slack OAuth2 token or revoked workspace access
  - Missing default channel configuration depending on credential/node setup
- **Sub-workflow reference:** None.

---

## Block 5 — Payment Monitoring and Follow-up

### Overview
This block waits, rechecks the HubSpot deal stage, and decides whether to close the process, send a reminder, or escalate. It relies on a specific paid stage value in HubSpot.

### Nodes Involved
- `⏳ Wait - 7 Day Payment Window`
- `🔍 HTTP - Recheck Deal Stage`
- `🔀 IF - Payment Received?`
- `✅ Slack - Payment Confirmed`
- `📧 Gmail - Follow-up Email #1`
- `⏳ Wait - 5 More Days`
- `🔍 HTTP - Final Payment Check`
- `🔀 IF - Still Unpaid? (Escalate)`
- `🚨 Slack - Escalation Alert`
- `📓 Notion - Flag as Overdue`
- `✅ Slack - Late Payment Confirmed`

### Node Details

#### 13. `⏳ Wait - 7 Day Payment Window`
- **Type and technical role:** Wait node; pauses execution for 7 days.
- **Configuration choices:** Unit `days`, amount `7`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `💬 Slack - Invoice Sent Alert`; output to `🔍 HTTP - Recheck Deal Stage`.
- **Version-specific requirements:** `typeVersion: 1`; requires n8n execution persistence support for long waits.
- **Edge cases or potential failure types:**
  - Wait jobs may fail if execution data retention or queue setup is not suited for long-running workflows
  - Timezone expectations are not explicitly managed
- **Sub-workflow reference:** None.

#### 14. `🔍 HTTP - Recheck Deal Stage`
- **Type and technical role:** HTTP Request node; reloads current HubSpot deal stage.
- **Configuration choices:** GET request using original `deal_id`; requests:
  - `dealstage`
  - `amount`
  - `hs_deal_stage_probability`
- **Key expressions or variables used:**
  - `{{ $('📄 Code - Build Invoice + HTML').item.json.deal_id }}`
- **Input and output connections:** Input from `⏳ Wait - 7 Day Payment Window`; output to `🔀 IF - Payment Received?`.
- **Version-specific requirements:** `typeVersion: 4`.
- **Edge cases or potential failure types:**
  - Missing auth configuration in node definition if not manually added
  - Deal may have been archived or permissions changed
- **Sub-workflow reference:** None.

#### 15. `🔀 IF - Payment Received?`
- **Type and technical role:** IF node; checks whether payment has been recorded.
- **Configuration choices:** Condition: `properties.dealstage == closedwon_paid`.
- **Key expressions or variables used:**
  - `{{ $json.properties.dealstage }}`
- **Input and output connections:** Input from `🔍 HTTP - Recheck Deal Stage`; true branch to `✅ Slack - Payment Confirmed`, false branch to `📧 Gmail - Follow-up Email #1`.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - `closedwon_paid` is a custom assumption and must match your actual HubSpot paid-stage internal value
  - If payment is tracked elsewhere, this check will not work
- **Sub-workflow reference:** None.

#### 16. `✅ Slack - Payment Confirmed`
- **Type and technical role:** Slack node; posts payment confirmation message.
- **Configuration choices:** Sends a celebratory message with invoice details and amount received.
- **Key expressions or variables used:** References invoice data from `📄 Code - Build Invoice + HTML`.
- **Input and output connections:** Input from true branch of `🔀 IF - Payment Received?`; no downstream nodes.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:** Slack auth/channel issues.
- **Sub-workflow reference:** None.

#### 17. `📧 Gmail - Follow-up Email #1`
- **Type and technical role:** Gmail node; sends first reminder email if payment is not yet confirmed.
- **Configuration choices:** HTML email reminding the client of the invoice and due date.
- **Key expressions or variables used:** Uses contact name, invoice number, amount, and due date from `📄 Code - Build Invoice + HTML`.
- **Input and output connections:** Input from false branch of `🔀 IF - Payment Received?`; output to `⏳ Wait - 5 More Days`.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Same Gmail/email validity concerns as the initial send
  - No threading/reply handling configured
- **Sub-workflow reference:** None.

#### 18. `⏳ Wait - 5 More Days`
- **Type and technical role:** Wait node; second delay before escalation check.
- **Configuration choices:** Unit `days`, amount `5`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `📧 Gmail - Follow-up Email #1`; output to `🔍 HTTP - Final Payment Check`.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** Same long-wait persistence concerns.
- **Sub-workflow reference:** None.

#### 19. `🔍 HTTP - Final Payment Check`
- **Type and technical role:** HTTP Request node; final verification of payment status in HubSpot.
- **Configuration choices:** GET request for the original deal; requests:
  - `dealstage`
  - `amount`
- **Key expressions or variables used:**
  - `{{ $('📄 Code - Build Invoice + HTML').item.json.deal_id }}`
- **Input and output connections:** Input from `⏳ Wait - 5 More Days`; output to `🔀 IF - Still Unpaid? (Escalate)`.
- **Version-specific requirements:** `typeVersion: 4`.
- **Edge cases or potential failure types:** Same HubSpot API auth/rate-limit/object existence concerns.
- **Sub-workflow reference:** None.

#### 20. `🔀 IF - Still Unpaid? (Escalate)`
- **Type and technical role:** IF node; branches between overdue escalation and late-payment confirmation.
- **Configuration choices:** Condition checks `properties.dealstage != closedwon_paid`.
- **Key expressions or variables used:**
  - `{{ $json.properties.dealstage }}`
- **Input and output connections:** Input from `🔍 HTTP - Final Payment Check`.
  - True branch: `🚨 Slack - Escalation Alert` and `📓 Notion - Flag as Overdue`
  - False branch: `✅ Slack - Late Payment Confirmed`
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** Same stage-ID mismatch risk.
- **Sub-workflow reference:** None.

#### 21. `🚨 Slack - Escalation Alert`
- **Type and technical role:** Slack node; internal overdue alert.
- **Configuration choices:** Posts a high-visibility message with invoice, client, amount, email, due date, and manual action request.
- **Key expressions or variables used:** References invoice data from `📄 Code - Build Invoice + HTML`.
- **Input and output connections:** Input from true branch of `🔀 IF - Still Unpaid? (Escalate)`; no downstream nodes.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:** Slack auth/channel issues.
- **Sub-workflow reference:** None.

#### 22. `📓 Notion - Flag as Overdue`
- **Type and technical role:** Notion node; intended to update the invoice record as overdue.
- **Configuration choices:** Operation set to `update`, but no visible target page/database properties are configured in the supplied JSON.
- **Key expressions or variables used:** None configured in the provided export.
- **Input and output connections:** Input from true branch of `🔀 IF - Still Unpaid? (Escalate)`; no downstream nodes.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Incomplete configuration as provided
  - Missing page/database ID and property mappings
  - Notion permissions issues
- **Sub-workflow reference:** None.

#### 23. `✅ Slack - Late Payment Confirmed`
- **Type and technical role:** Slack node; posts when payment is received after the reminder cycle.
- **Configuration choices:** Sends a “late payment received” message and reminds the team to update HubSpot manually if needed.
- **Key expressions or variables used:** References invoice data from `📄 Code - Build Invoice + HTML`.
- **Input and output connections:** Input from false branch of `🔀 IF - Still Unpaid? (Escalate)`; no downstream nodes.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:** Slack auth/channel issues.
- **Sub-workflow reference:** None.

---

## Block 6 — Error Handling

### Overview
This independent branch catches workflow failures and sends a structured Slack alert containing execution metadata. It provides a simple operational monitoring layer for manual response.

### Nodes Involved
- `⚡ Error Trigger`
- `🚨 Slack - Workflow Error Alert`

### Node Details

#### 24. `⚡ Error Trigger`
- **Type and technical role:** Error Trigger node; starts an execution when another workflow execution fails.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** Exposes execution context in the output payload.
- **Input and output connections:** No input; output to `🚨 Slack - Workflow Error Alert`.
- **Version-specific requirements:** `typeVersion: 1`; works only when n8n error workflow behavior is correctly configured.
- **Edge cases or potential failure types:**
  - Depending on deployment and workflow settings, may require explicit error-workflow association
  - If Slack alerting also fails, the error can remain silent
- **Sub-workflow reference:** None.

#### 25. `🚨 Slack - Workflow Error Alert`
- **Type and technical role:** Slack node; posts failure diagnostics.
- **Configuration choices:** Markdown-formatted message includes:
  - error message
  - failed node
  - workflow name
  - execution ID
  - formatted timestamp
- **Key expressions or variables used:**
  - `{{ $json.execution.error.message }}`
  - `{{ $json.execution.lastNodeExecuted }}`
  - `{{ $json.workflow.name }}`
  - `{{ $json.execution.id }}`
  - `{{ $now.format('yyyy-MM-dd HH:mm:ss') }}`
- **Input and output connections:** Input from `⚡ Error Trigger`; no downstream nodes.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:** Slack auth/channel issues; some error payload fields may differ by n8n version.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 🎯 HubSpot - Deal Trigger | HubSpot Trigger | Entry point on HubSpot deal property change |  | 🔀 IF - Is Deal Closed Won? | ##  Trigger & Data Collection<br>Triggers on deal stage change.<br>Fetches deal + contact details from HubSpot. |
| 🔀 IF - Is Deal Closed Won? | IF | Filters only stage changes to `closedwon` | 🎯 HubSpot - Deal Trigger | 🏷️ HTTP - Get Deal Details | ##  Trigger & Data Collection<br>Triggers on deal stage change.<br>Fetches deal + contact details from HubSpot. |
| 🏷️ HTTP - Get Deal Details | HTTP Request | Retrieves full deal details from HubSpot | 🔀 IF - Is Deal Closed Won? | 🔗 HTTP - Get Deal Associations | ##  Trigger & Data Collection<br>Triggers on deal stage change.<br>Fetches deal + contact details from HubSpot. |
| 🔗 HTTP - Get Deal Associations | HTTP Request | Retrieves associated contacts for the deal | 🏷️ HTTP - Get Deal Details | ⚙️ Code - Extract Contact ID | ##  Trigger & Data Collection<br>Triggers on deal stage change.<br>Fetches deal + contact details from HubSpot. |
| ⚙️ Code - Extract Contact ID | Code | Validates associations and extracts billing contact/deal fields | 🔗 HTTP - Get Deal Associations | 👤 HTTP - Get Contact Details | ##  Trigger & Data Collection<br>Triggers on deal stage change.<br>Fetches deal + contact details from HubSpot. |
| 👤 HTTP - Get Contact Details | HTTP Request | Retrieves customer contact details from HubSpot | ⚙️ Code - Extract Contact ID | 📄 Code - Build Invoice + HTML | ##  Trigger & Data Collection<br>Triggers on deal stage change.<br>Fetches deal + contact details from HubSpot. |
| 📄 Code - Build Invoice + HTML | Code | Generates invoice metadata and HTML document | 👤 HTTP - Get Contact Details | 🖨️ HTTP - Generate PDF | ## Invoice Generation<br>Builds invoice data and creates styled HTML. |
| 🖨️ HTTP - Generate PDF | HTTP Request | Converts HTML invoice to PDF file | 📄 Code - Build Invoice + HTML | 📊 Google Sheets - Log Invoice | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| 📊 Google Sheets - Log Invoice | Google Sheets | Appends invoice metadata to tracking sheet | 🖨️ HTTP - Generate PDF | 📓 Notion - Create Invoice Record | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| 📓 Notion - Create Invoice Record | Notion | Creates invoice record/page in Notion | 📊 Google Sheets - Log Invoice | 📧 Gmail - Send Invoice Email | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| 📧 Gmail - Send Invoice Email | Gmail | Sends initial invoice email to customer | 📓 Notion - Create Invoice Record | 💬 Slack - Invoice Sent Alert | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| 💬 Slack - Invoice Sent Alert | Slack | Notifies team that invoice was sent | 📧 Gmail - Send Invoice Email | ⏳ Wait - 7 Day Payment Window | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| ⏳ Wait - 7 Day Payment Window | Wait | Delays execution before first payment check | 💬 Slack - Invoice Sent Alert | 🔍 HTTP - Recheck Deal Stage | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| 🔍 HTTP - Recheck Deal Stage | HTTP Request | Re-checks HubSpot deal stage after 7 days | ⏳ Wait - 7 Day Payment Window | 🔀 IF - Payment Received? | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| 🔀 IF - Payment Received? | IF | Decides between payment confirmation and reminder flow | 🔍 HTTP - Recheck Deal Stage | ✅ Slack - Payment Confirmed; 📧 Gmail - Follow-up Email #1 | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| ✅ Slack - Payment Confirmed | Slack | Alerts team that payment was confirmed on first check | 🔀 IF - Payment Received? |  | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| 📧 Gmail - Follow-up Email #1 | Gmail | Sends first overdue reminder email | 🔀 IF - Payment Received? | ⏳ Wait - 5 More Days | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| ⏳ Wait - 5 More Days | Wait | Adds second delay before final payment check | 📧 Gmail - Follow-up Email #1 | 🔍 HTTP - Final Payment Check | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| 🔍 HTTP - Final Payment Check | HTTP Request | Performs final HubSpot payment-status check | ⏳ Wait - 5 More Days | 🔀 IF - Still Unpaid? (Escalate) | ##  Escalation & Errors<br>Handles overdue invoices and workflow failures.<br>Alerts team and updates records. |
| 🔀 IF - Still Unpaid? (Escalate) | IF | Branches to overdue escalation or late payment confirmation | 🔍 HTTP - Final Payment Check | 🚨 Slack - Escalation Alert; 📓 Notion - Flag as Overdue; ✅ Slack - Late Payment Confirmed | ##  Escalation & Errors<br>Handles overdue invoices and workflow failures.<br>Alerts team and updates records. |
| 🚨 Slack - Escalation Alert | Slack | Posts overdue invoice escalation alert | 🔀 IF - Still Unpaid? (Escalate) |  | ##  Escalation & Errors<br>Handles overdue invoices and workflow failures.<br>Alerts team and updates records. |
| 📓 Notion - Flag as Overdue | Notion | Intended to update Notion record as overdue | 🔀 IF - Still Unpaid? (Escalate) |  | ##  Escalation & Errors<br>Handles overdue invoices and workflow failures.<br>Alerts team and updates records. |
| ✅ Slack - Late Payment Confirmed | Slack | Posts late-payment confirmation after reminder cycle | 🔀 IF - Still Unpaid? (Escalate) |  | ##  Escalation & Errors<br>Handles overdue invoices and workflow failures.<br>Alerts team and updates records. |
| ⚡ Error Trigger | Error Trigger | Starts error-monitoring branch when workflow fails |  | 🚨 Slack - Workflow Error Alert | ## Error Handling<br>Captures any workflow failure<br>and sends alert to Slack for quick action. |
| 🚨 Slack - Workflow Error Alert | Slack | Sends workflow failure diagnostics to Slack | ⚡ Error Trigger |  | ## Error Handling<br>Captures any workflow failure<br>and sends alert to Slack for quick action. |
| 📋 SETUP INSTRUCTIONS | Sticky Note | Documentation note |  |  | # Automated Invoice & Follow-Up System<br><br>### How it works<br>This workflow triggers when a HubSpot deal is marked as Closed Won. It fetches deal and contact data, generates a styled invoice, converts it into a PDF, logs it in internal systems, and sends it to the client.<br><br>After sending, the system tracks payment status. If unpaid, it sends reminders and escalates overdue invoices automatically.<br><br>### Setup steps<br>1. Connect HubSpot trigger and API credentials<br>2. Add HTML2PDF API key<br>3. Set Google Sheets and Notion database<br>4. Connect Gmail and Slack<br>5. Update payment stage ID in IF nodes<br>6. Activate workflow<br><br>### Customization tips<br>- Edit invoice design in Code node<br>- Adjust wait durations for follow-ups<br>- Add more reminders if needed |
| Sticky Note7 | Sticky Note | Documentation note |  |  | ##  Trigger & Data Collection<br>Triggers on deal stage change.<br>Fetches deal + contact details from HubSpot. |
| Sticky Note8 | Sticky Note | Documentation note |  |  | ## Invoice Generation<br>Builds invoice data and creates styled HTML. |
| Sticky Note9 | Sticky Note | Documentation note |  |  | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| Sticky Note10 | Sticky Note | Documentation note |  |  | ## Send & Store Invoice<br>Converts to PDF, logs in systems,<br>emails client, and alerts team. |
| Sticky Note11 | Sticky Note | Documentation note |  |  | ##  Escalation & Errors<br>Handles overdue invoices and workflow failures.<br>Alerts team and updates records. |
| Sticky Note12 | Sticky Note | Documentation note |  |  | ## Error Handling<br>Captures any workflow failure<br>and sends alert to Slack for quick action. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Generate and track invoices with HubSpot, Gmail, Slack, Sheets, and Notion`.

2. **Add a HubSpot Trigger node**
   - Node type: `HubSpot Trigger`
   - Name: `🎯 HubSpot - Deal Trigger`
   - Configure event: `deal.propertyChange`
   - Connect valid HubSpot credentials
   - Ensure HubSpot webhook/app permissions cover CRM deals

3. **Add an IF node after the trigger**
   - Name: `🔀 IF - Is Deal Closed Won?`
   - Add two string conditions:
     - `{{$json.propertyName}}` equals `dealstage`
     - `{{$json.propertyValue}}` equals `closedwon`
   - Connect the trigger output to this IF node
   - Use the true output branch only

4. **Add an HTTP Request node to fetch deal details**
   - Name: `🏷️ HTTP - Get Deal Details`
   - Method: `GET`
   - URL:  
     `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.objectId }}?properties=dealname,amount,dealstage,closedate,hubspot_owner_id,description`
   - Authentication: generic credential type using HTTP Header Auth
   - Configure Authorization header with your HubSpot private app token, for example `Bearer YOUR_TOKEN`
   - Connect IF true output to this node

5. **Add an HTTP Request node to fetch deal-contact associations**
   - Name: `🔗 HTTP - Get Deal Associations`
   - Method: `GET`
   - URL:  
     `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.id }}/associations/contacts`
   - Use the same HubSpot header auth
   - Connect from `🏷️ HTTP - Get Deal Details`

6. **Add a Code node to extract the contact ID**
   - Name: `⚙️ Code - Extract Contact ID`
   - Paste logic that:
     - reads `results` from current input
     - throws an error if there are no associated contacts
     - takes the first contact ID
     - reads deal details from `🏷️ HTTP - Get Deal Details`
     - returns `contactId`, `dealId`, `dealName`, `amount`, and `closeDate`
   - Connect from `🔗 HTTP - Get Deal Associations`

7. **Add an HTTP Request node to fetch contact details**
   - Name: `👤 HTTP - Get Contact Details`
   - Method: `GET`
   - URL:  
     `https://api.hubapi.com/crm/v3/objects/contacts/{{ $json.contactId }}?properties=firstname,lastname,email,company,phone,address`
   - Use the same HubSpot authentication
   - Connect from `⚙️ Code - Extract Contact ID`

8. **Add a Code node to build invoice data and HTML**
   - Name: `📄 Code - Build Invoice + HTML`
   - Implement logic to:
     - read current contact properties
     - read previous deal data from `⚙️ Code - Extract Contact ID`
     - generate invoice number like `INV-2026-1234`
     - generate issue date as current date
     - generate due date as current date + 30 days
     - parse amount and produce `formatted_amount`
     - create complete HTML invoice content
   - Return these fields:
     - `invoice_number`
     - `issue_date`
     - `due_date`
     - `amount`
     - `formatted_amount`
     - `deal_name`
     - `contact_name`
     - `contact_email`
     - `company`
     - `phone`
     - `deal_id`
     - `contact_id`
     - `html_content`
   - Update all placeholder branding:
     - company name
     - address
     - support email
     - phone number
     - bank details
   - Connect from `👤 HTTP - Get Contact Details`

9. **Add an HTTP Request node to generate the PDF**
   - Name: `🖨️ HTTP - Generate PDF`
   - Method: `POST`
   - URL: `https://api.html2pdf.app/v1/generate`
   - Enable body sending
   - Send fields:
     - `html` = `{{$json.html_content}}`
     - `apiKey` = your html2pdf.app API key
     - `media_type` = `print`
     - `format` = `A4`
     - `landscape` = `false`
   - Configure response format as `File`
   - Set binary property name: `invoice_pdf`
   - Connect from `📄 Code - Build Invoice + HTML`

10. **Add a Google Sheets node to log invoice metadata**
    - Name: `📊 Google Sheets - Log Invoice`
    - Operation: `Append`
    - Set Google credentials
    - Choose spreadsheet ID
    - Sheet name: typically `Sheet1` or your preferred sheet
    - Create headers in the sheet:
      - Email
      - Amount
      - Status
      - Company
      - Due Date
      - Deal Name
      - Issue Date
      - Client Name
      - Invoice Number
      - HubSpot Deal ID
    - Map values from `📄 Code - Build Invoice + HTML`
    - Set Status to `Sent - Awaiting Payment`
    - Connect from `🖨️ HTTP - Generate PDF`

11. **Add a Notion node to create an invoice record**
    - Name: `📓 Notion - Create Invoice Record`
    - Connect Notion credentials
    - Configure it to create either:
      - a page in a parent page, or
      - preferably a database record in an invoice database
    - Set title to:  
      `{{$('📄 Code - Build Invoice + HTML').item.json.invoice_number}} — {{$('📄 Code - Build Invoice + HTML').item.json.contact_name}}`
    - If using a database, also store invoice number, amount, status, due date, deal ID, and contact email
    - Connect from `📊 Google Sheets - Log Invoice`

12. **Add a Gmail node to send the invoice email**
    - Name: `📧 Gmail - Send Invoice Email`
    - Connect Gmail OAuth2 credentials
    - To: `{{$('📄 Code - Build Invoice + HTML').item.json.contact_email}}`
    - Subject:  
      `Invoice {{$('📄 Code - Build Invoice + HTML').item.json.invoice_number}} — ${{$('📄 Code - Build Invoice + HTML').item.json.formatted_amount}} Due {{$('📄 Code - Build Invoice + HTML').item.json.due_date}}`
    - Message: HTML email including:
      - customer greeting
      - invoice number
      - amount due
      - due date
      - payment instructions
    - Important: if you want the PDF actually attached, explicitly configure attachment support using binary property `invoice_pdf`
    - Connect from `📓 Notion - Create Invoice Record`

13. **Add a Slack node for the “invoice sent” alert**
    - Name: `💬 Slack - Invoice Sent Alert`
    - Connect Slack OAuth2 credentials
    - Send a Markdown-enabled message containing:
      - invoice number
      - client name
      - company
      - amount
      - deal
      - due date
      - note about follow-up in 7 days
    - Connect from `📧 Gmail - Send Invoice Email`

14. **Add a Wait node for the first payment window**
    - Name: `⏳ Wait - 7 Day Payment Window`
    - Wait time: `7 days`
    - Connect from `💬 Slack - Invoice Sent Alert`

15. **Add an HTTP Request node to recheck deal stage**
    - Name: `🔍 HTTP - Recheck Deal Stage`
    - Method: `GET`
    - URL:  
      `https://api.hubapi.com/crm/v3/objects/deals/{{ $('📄 Code - Build Invoice + HTML').item.json.deal_id }}?properties=dealstage,amount,hs_deal_stage_probability`
    - Use HubSpot auth
    - Connect from `⏳ Wait - 7 Day Payment Window`

16. **Add an IF node to test whether payment was received**
    - Name: `🔀 IF - Payment Received?`
    - Condition: `{{$json.properties.dealstage}}` equals `closedwon_paid`
    - Replace `closedwon_paid` with your real paid-stage internal value in HubSpot
    - Connect from `🔍 HTTP - Recheck Deal Stage`

17. **Add a Slack node for immediate payment confirmation**
    - Name: `✅ Slack - Payment Confirmed`
    - Connect from the true output of `🔀 IF - Payment Received?`
    - Configure a success message with invoice number, customer, company, and amount

18. **Add a Gmail node for the first reminder**
    - Name: `📧 Gmail - Follow-up Email #1`
    - Connect from the false output of `🔀 IF - Payment Received?`
    - Send a reminder email to the same customer email
    - Mention:
      - invoice number
      - amount
      - due date
      - polite ignore-if-already-paid notice

19. **Add a second Wait node**
    - Name: `⏳ Wait - 5 More Days`
    - Wait time: `5 days`
    - Connect from `📧 Gmail - Follow-up Email #1`

20. **Add an HTTP Request node for the final payment check**
    - Name: `🔍 HTTP - Final Payment Check`
    - Method: `GET`
    - URL:  
      `https://api.hubapi.com/crm/v3/objects/deals/{{ $('📄 Code - Build Invoice + HTML').item.json.deal_id }}?properties=dealstage,amount`
    - Use HubSpot auth
    - Connect from `⏳ Wait - 5 More Days`

21. **Add a final IF node for escalation decision**
    - Name: `🔀 IF - Still Unpaid? (Escalate)`
    - Condition: `{{$json.properties.dealstage}}` not equal to `closedwon_paid`
    - True means still unpaid; false means paid late
    - Connect from `🔍 HTTP - Final Payment Check`

22. **Add a Slack node for overdue escalation**
    - Name: `🚨 Slack - Escalation Alert`
    - Connect from the true output of `🔀 IF - Still Unpaid? (Escalate)`
    - Configure an alert containing:
      - invoice number
      - client
      - amount
      - email
      - due date
      - instruction for manual follow-up

23. **Add a Notion node to flag the invoice as overdue**
    - Name: `📓 Notion - Flag as Overdue`
    - Operation: `Update`
    - Connect from the same true output of `🔀 IF - Still Unpaid? (Escalate)`
    - Configure it to update the previously created page/database record
    - You need a reliable record ID strategy; options include:
      - storing the Notion page ID from the create step
      - searching by invoice number before updating
    - Update a property such as `Status = OVERDUE`

24. **Add a Slack node for late payment confirmation**
    - Name: `✅ Slack - Late Payment Confirmed`
    - Connect from the false output of `🔀 IF - Still Unpaid? (Escalate)`
    - Configure a message noting payment arrived after follow-up

25. **Create a separate error-handling branch**
    - Add node: `⚡ Error Trigger`
    - This starts when a workflow execution fails

26. **Add a Slack node after the Error Trigger**
    - Name: `🚨 Slack - Workflow Error Alert`
    - Connect from `⚡ Error Trigger`
    - Use expressions:
      - error message: `{{$json.execution.error.message}}`
      - failed node: `{{$json.execution.lastNodeExecuted}}`
      - workflow name: `{{$json.workflow.name}}`
      - execution ID: `{{$json.execution.id}}`
      - timestamp: `{{$now.format('yyyy-MM-dd HH:mm:ss')}}`
    - Send to an operations or alerts channel

27. **Add sticky notes for maintainability**
    - Setup instructions
    - Trigger & data collection
    - Invoice generation
    - Send & store invoice
    - Escalation & errors
    - Error handling

28. **Validate credentials**
    - HubSpot:
      - Trigger credentials
      - HTTP Header Auth for API requests
    - html2pdf.app:
      - API key in request body
    - Google Sheets:
      - OAuth2 or service account with spreadsheet access
    - Notion:
      - Integration token and page/database sharing
    - Gmail:
      - OAuth2 with send permissions
    - Slack:
      - OAuth2 and target channel access

29. **Test with a non-production deal**
    - Move a HubSpot deal into the target won stage
    - Confirm:
      - invoice HTML is generated
      - PDF service responds
      - spreadsheet row is added
      - Notion record is created
      - email is sent
      - Slack alert posts

30. **Test delayed follow-up behavior**
    - Temporarily shorten Wait nodes to minutes during testing
    - Verify both paths:
      - paid before reminder
      - unpaid leading to reminder and escalation

31. **Activate the workflow**
    - Ensure all placeholders have been replaced:
      - `REPLACE_HTML2PDF_API_KEY`
      - `REPLACE_GOOGLE_SHEET_ID`
      - Notion parent/database target
      - paid stage internal value
      - company branding in HTML/email content

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated Invoice & Follow-Up System | Internal workflow purpose |
| This workflow triggers when a HubSpot deal is marked as Closed Won. It fetches deal and contact data, generates a styled invoice, converts it into a PDF, logs it in internal systems, and sends it to the client. | Setup note |
| After sending, the system tracks payment status. If unpaid, it sends reminders and escalates overdue invoices automatically. | Setup note |
| Setup steps: Connect HubSpot trigger and API credentials; Add HTML2PDF API key; Set Google Sheets and Notion database; Connect Gmail and Slack; Update payment stage ID in IF nodes; Activate workflow | Setup note |
| Customization tips: Edit invoice design in Code node; Adjust wait durations for follow-ups; Add more reminders if needed | Setup note |
| Important implementation gap: the Gmail node text says the invoice PDF is attached, but the exported node configuration does not include an attachment setting. | Operational note |
| Important implementation gap: `👤 HTTP - Get Contact Details`, `🔍 HTTP - Recheck Deal Stage`, and `🔍 HTTP - Final Payment Check` do not show explicit credentials in the export and should be checked manually. | Operational note |
| Important implementation gap: both Notion nodes are incomplete in the export and require full target/page/database/update configuration before use. | Operational note |
| Important implementation gap: invoice number generation is random and does not guarantee uniqueness. | Design note |
| Important implementation gap: `closedwon` and `closedwon_paid` must match your HubSpot internal stage values, not just the display labels. | HubSpot configuration note |