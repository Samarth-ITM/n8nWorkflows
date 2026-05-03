Send overdue invoice payment reminders with iFirma, Gmail, PostGrid and Slack

https://n8nworkflows.xyz/workflows/send-overdue-invoice-payment-reminders-with-ifirma--gmail--postgrid-and-slack-14310


# Send overdue invoice payment reminders with iFirma, Gmail, PostGrid and Slack

# 1. Workflow Overview

This workflow automates overdue invoice follow-up for invoices issued through **iFirma**. It runs on a schedule, retrieves unpaid or overdue invoices from iFirma using a custom HMAC-SHA1 authentication scheme, enriches them with contractor details, and then routes each invoice into one of three escalation paths:

- **Payment reminder**
- **Pre-trial summon / notice of intent to pursue legal action**
- **Court summons / entering legal route**

Depending on the branch, it sends messages through:
- **iFirma native reminder sending**
- **Gmail**
- **PostGrid physical mail**
- **Slack notifications**

It is intended for businesses that want structured receivables follow-up with configurable timing thresholds before and after the due date.

## 1.1 Scheduling and Core Configuration

The workflow starts on a daily schedule, then loads:
- iFirma login and API key
- escalation thresholds
- sender company details used in notices and postal mail

## 1.2 Invoice Retrieval from iFirma

The workflow prepares the iFirma invoice list endpoint, generates the required HMAC-SHA1 authorization header in a Code node, fetches invoices, checks whether the API returned an error in the response body, and then splits the invoice list into individual items.

## 1.3 Invoice Filtering and Endpoint Mapping

Only invoices that are due today, overdue, or at the configured reminder lead time are kept. Each invoice type is mapped to the proper iFirma API endpoint needed later for sending reminders.

## 1.4 Contractor Enrichment

Because invoice payloads do not contain full contractor details, the workflow deduplicates contractors, fetches contractor records from iFirma, checks for errors, extracts the contractor object, and merges contractor data back into each invoice.

## 1.5 Escalation Routing

Each enriched invoice is evaluated against three timing rules:
- due today or X days before due date → payment reminder
- exactly Y days after due date → pre-trial summon
- exactly Z days after due date → court/legal route notice

## 1.6 Outbound Notification Paths

Each route has its own delivery behavior:
- **Payment reminder:** send through iFirma and notify Slack
- **Pre-trial summon:** notify Slack, generate HTML, optionally send by Gmail and/or PostGrid
- **Court/legal route notice:** notify Slack, generate HTML, optionally send by Gmail and/or PostGrid

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduling and Base Configuration

### Overview
This block initializes the workflow once per day and centralizes business configuration. It stores iFirma credentials, timing thresholds, and sender company information used in later email and letter content.

### Nodes Involved
- Schedule Daily Check
- Configuration
- Your Company Details

### Node Details

#### 1. Schedule Daily Check
- **Type / role:** `Schedule Trigger`; entry point for daily execution.
- **Configuration choices:** Configured with a 24-hour interval. The node note says it runs every day at 9:00 AM, but the stored rule is interval-based rather than explicit cron-at-9 syntax.
- **Key expressions / variables:** None.
- **Input / output connections:** Entry node → outputs to **Configuration**.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:**
  - If workflow timezone is not configured as expected, execution time may differ.
  - The note and actual schedule settings may diverge.
- **Sub-workflow reference:** None.

#### 2. Configuration
- **Type / role:** `Set`; stores iFirma auth inputs and escalation timing parameters.
- **Configuration choices:** Defines:
  - `Email/Login`
  - `API Key Invoice`
  - `X days before due date` = 7
  - `Y days after due date` = 7
  - `Z days after due date` = 14
- **Key expressions / variables:** Later referenced using:
  - `$('Configuration').first().json["Email/Login"]`
  - `$('Configuration').first().json["API Key Invoice"]`
  - threshold fields
- **Input / output connections:** Input from **Schedule Daily Check** → output to **Your Company Details**.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:**
  - Blank login/API key breaks all iFirma calls.
  - Non-sensical threshold values can cause overlapping or skipped escalation windows.
  - If `Y >= Z`, the legal escalation sequence becomes logically inconsistent.
- **Sub-workflow reference:** None.

#### 3. Your Company Details
- **Type / role:** `Set`; stores sender identity and banking/address details for legal notices and PostGrid mail.
- **Configuration choices:** Includes:
  - company name
  - email
  - phone
  - country code
  - city
  - street
  - postal code
  - SWIFT code
  - bank name
  - bank account number
  - tax ID
- **Key expressions / variables:** Referenced throughout HTML content and PostGrid requests via `$('Your Company Details').first().json[...]`.
- **Input / output connections:** Input from **Configuration** → output to **URL to Fetch Invoices**.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:**
  - Missing address data can cause PostGrid request validation failures.
  - Missing bank/contact data leads to incomplete legal notice content.
- **Sub-workflow reference:** None.

---

## Block 2 — Invoice Retrieval from iFirma

### Overview
This block builds the invoices API request, computes the custom iFirma HMAC authorization header, sends the request, and handles API errors returned inside the response body.

### Nodes Involved
- URL to Fetch Invoices
- Encode API Key
- Fetch Invoices from System
- Contains Error?
- Error Fetching Invoices
- Extract Invoices

### Node Details

#### 4. URL to Fetch Invoices
- **Type / role:** `Set`; defines the iFirma invoice list endpoint and request body placeholder.
- **Configuration choices:**
  - `url` = `https://www.ifirma.pl/iapi/faktury.json`
  - `requestBody` = empty string
- **Key expressions / variables:** `url`, `requestBody`
- **Input / output connections:** Input from **Your Company Details** → output to **Encode API Key**.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:** Incorrect URL breaks auth signature and request.
- **Sub-workflow reference:** None.

#### 5. Encode API Key
- **Type / role:** `Code`; generates iFirma `Authentication` header using HMAC-SHA1.
- **Configuration choices:** Reads:
  - API key from **Configuration**
  - login from **Configuration**
  - `url` and `requestBody` from upstream input  
  Constructs message as `(url + userLogin + keyName + requestBody).trim()` where `keyName = "faktura"`.
- **Key expressions / variables:**
  - `$('Configuration').first().json["API Key Invoice"]`
  - `$('Configuration').first().json["Email/Login"]`
  - outputs `{ authHeader }`
- **Input / output connections:** Input from **URL to Fetch Invoices** → output to **Fetch Invoices from System**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - Invalid hex API key causes broken HMAC.
  - Any mismatch in signing string format causes authentication rejection.
  - The SHA-1 implementation is custom JavaScript, so code edits are risky.
- **Sub-workflow reference:** None.

#### 6. Fetch Invoices from System
- **Type / role:** `HTTP Request`; retrieves invoices from iFirma.
- **Configuration choices:**
  - URL from **URL to Fetch Invoices**
  - Sends query parameter `status=przeterminowane,nieoplacone,oplaconeCzesciowo`
  - Sends `Authentication` header from the previous node
  - Pagination settings include parameter `strona=1`
- **Key expressions / variables:**
  - `{{ $('URL to Fetch Invoices').first().json.url }}`
  - `{{ $json.authHeader }}`
- **Input / output connections:** Input from **Encode API Key** → output to **Contains Error?**
- **Version-specific requirements:** Type version 4.1.
- **Edge cases / failures:**
  - iFirma may return logical errors inside `response.Kod` rather than as HTTP failure.
  - Pagination appears only initialized to page 1; if n8n pagination is not fully configured for this API, additional pages may not be fetched.
  - Timeout or connectivity issues stop the run.
- **Sub-workflow reference:** None.

#### 7. Contains Error?
- **Type / role:** `If`; checks whether iFirma returned an application-level error.
- **Configuration choices:** Tests `{{ $json.response.Kod }}` greater than `0`.
- **Key expressions / variables:** `$json.response.Kod`
- **Input / output connections:**
  - Input from **Fetch Invoices from System**
  - True → **Error Fetching Invoices**
  - False → **Extract Invoices**
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:**
  - If the response structure changes and `response.Kod` is missing, strict number comparison may fail.
- **Sub-workflow reference:** None.

#### 8. Error Fetching Invoices
- **Type / role:** `Stop And Error`; terminates execution with a structured error object.
- **Configuration choices:** Returns:
  - message: `"Error fetching invoices"`
  - body: serialized `Fetch Invoices from System` response
- **Key expressions / variables:** `JSON.stringify($("Fetch Invoices from System").last().json.response)`
- **Input / output connections:** Input from **Contains Error?** true branch.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:**
  - If prior response is malformed, expression may fail.
- **Sub-workflow reference:** None.

#### 9. Extract Invoices
- **Type / role:** `Split Out`; expands `response.Wynik` invoice array into one item per invoice.
- **Configuration choices:** `fieldToSplitOut = response.Wynik`
- **Key expressions / variables:** `response.Wynik`
- **Input / output connections:** Input from **Contains Error?** false branch → output to **Filter Unpaid Invoices Due Today or Earlier**.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:**
  - If `response.Wynik` is empty, downstream nodes receive no items.
  - If structure differs, split fails.
- **Sub-workflow reference:** None.

---

## Block 3 — Invoice Filtering and Endpoint Mapping

### Overview
This block narrows invoices to those relevant for reminder flows and adds the correct iFirma sending endpoint based on invoice type.

### Nodes Involved
- Filter Unpaid Invoices Due Today or Earlier
- Assign Endpoint to Invoice Type

### Node Details

#### 10. Filter Unpaid Invoices Due Today or Earlier
- **Type / role:** `Filter`; keeps invoices either already due/overdue or exactly X days before due date.
- **Configuration choices:** OR logic:
  1. `TerminPlatnosci` before or equal to now
  2. `TerminPlatnosci` equals today minus configured `X days before due date`
- **Key expressions / variables:**
  - `{{new Date($json.TerminPlatnosci) }}`
  - `{{ $now }}`
  - `{{ $now.minus($('Configuration').first().json["X days before due date"], 'days').format('yyyy-MM-dd') }}`
- **Input / output connections:** Input from **Extract Invoices** → output to **Assign Endpoint to Invoice Type**.
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:**
  - There is a likely logical mismatch: “X days before due date” is compared to `today - X`, whereas semantically “before due date” usually implies `today + X = due date`.
  - Date parsing depends on `TerminPlatnosci` format.
  - Timezone differences can misclassify border cases.
- **Sub-workflow reference:** None.

#### 11. Assign Endpoint to Invoice Type
- **Type / role:** `Code`; maps invoice type `Rodzaj` to iFirma send endpoint.
- **Configuration choices:** Uses a hardcoded object map, e.g.:
  - `prz_faktura_kraj` → `fakturakraj`
  - `prz_faktura_proforma` → `fakturaproformakraj`
  - `prz_rachunek_kraj` → `rachunekkraj`
- **Key expressions / variables:**
  - input field `Rodzaj`
  - output field `endpoint`
- **Input / output connections:**
  - Input from **Filter Unpaid Invoices Due Today or Earlier**
  - Output to **Deduplicate Contractors**
  - Output to **Merge Invoice with contractor** input 0
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - If an unknown `Rodzaj` arrives, `endpoint` becomes undefined.
  - Undefined endpoint later breaks the reminder send URL.
- **Sub-workflow reference:** None.

---

## Block 4 — Contractor Enrichment and Merge

### Overview
This block avoids duplicate contractor lookups, fetches full contractor records from iFirma, validates those responses, and merges contractor data back into invoices.

### Nodes Involved
- Deduplicate Contractors
- URL to Fetch Contractor Information
- Encode API Key to Fetch Contractor Information
- Fetch Contractor Information
- Contains Error? 2
- Error Fetching Contractor Data
- Extract Contractor Data
- Merge Invoice with contractor

### Node Details

#### 12. Deduplicate Contractors
- **Type / role:** `Remove Duplicates`; ensures each contractor is fetched once.
- **Configuration choices:** Compares selected field `IdentyfikatorKontrahenta`.
- **Key expressions / variables:** `IdentyfikatorKontrahenta`
- **Input / output connections:** Input from **Assign Endpoint to Invoice Type** → output to **URL to Fetch Contractor Information**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - If invoices lack contractor ID, deduplication may collapse unrelated records or fail to help.
- **Sub-workflow reference:** None.

#### 13. URL to Fetch Contractor Information
- **Type / role:** `Set`; builds contractor lookup URL.
- **Configuration choices:**
  - `url` = `https://www.ifirma.pl/iapi/kontrahenci/id/{{$json.IdentyfikatorKontrahenta}}.json`
  - `requestBody` = empty string
- **Key expressions / variables:** `IdentyfikatorKontrahenta`
- **Input / output connections:** Input from **Deduplicate Contractors** → output to **Encode API Key to Fetch Contractor Information**.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:** Missing contractor ID causes invalid URL.
- **Sub-workflow reference:** None.

#### 14. Encode API Key to Fetch Contractor Information
- **Type / role:** `Code`; generates auth header for contractor lookup.
- **Configuration choices:** Same HMAC-SHA1 logic as invoice fetch.
- **Key expressions / variables:** Same login/API key references as earlier auth node.
- **Input / output connections:** Input from **URL to Fetch Contractor Information** → output to **Fetch Contractor Information**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - Same HMAC risks as other auth code nodes.
  - Contains debug `console.log`.
- **Sub-workflow reference:** None.

#### 15. Fetch Contractor Information
- **Type / role:** `HTTP Request`; retrieves contractor details from iFirma.
- **Configuration choices:**
  - Sends `Authentication` header
  - URL expression is configured as `{{ $("URL to Fetch Contractor Information").item.json }}` which appears incorrect because it references the whole JSON object instead of `.url`
- **Key expressions / variables:**
  - current auth header
  - intended contractor URL from prior node
- **Input / output connections:** Input from **Encode API Key to Fetch Contractor Information** → output to **Contains Error? 2**
- **Version-specific requirements:** Type version 4.3.
- **Edge cases / failures:**
  - Likely configuration bug: URL expression should probably reference `.json.url`; as written it may serialize incorrectly and fail.
  - iFirma logical errors can still arrive in response body.
- **Sub-workflow reference:** None.

#### 16. Contains Error? 2
- **Type / role:** `If`; checks contractor lookup response for iFirma application error.
- **Configuration choices:** `response.Kod > 0`
- **Key expressions / variables:** `$json.response.Kod`
- **Input / output connections:**
  - True → **Error Fetching Contractor Data**
  - False → **Extract Contractor Data**
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:** Same response-shape concerns as the first error check.
- **Sub-workflow reference:** None.

#### 17. Error Fetching Contractor Data
- **Type / role:** `Stop And Error`; aborts workflow on contractor fetch error.
- **Configuration choices:** Returns:
  - message: `"Error fetching contractor data"`
  - body: serialized response from **Fetch Invoices from System**
- **Key expressions / variables:** References invoice fetch response rather than contractor fetch response.
- **Input / output connections:** Input from **Contains Error? 2** true branch.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:**
  - Likely misreference: it should probably serialize contractor response, not invoice response.
- **Sub-workflow reference:** None.

#### 18. Extract Contractor Data
- **Type / role:** `Set`; extracts `response.Wynik[0]` as the contractor object.
- **Configuration choices:** Raw JSON mode using `JSON.stringify($input.item.json.response.Wynik[0], null, 2)`.
- **Key expressions / variables:** `response.Wynik[0]`
- **Input / output connections:** Input from **Contains Error? 2** false branch → output to **Merge Invoice with contractor** input 1.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:**
  - Empty result arrays produce undefined contractor data.
- **Sub-workflow reference:** None.

#### 19. Merge Invoice with contractor
- **Type / role:** `Merge`; combines invoice items with contractor items by matching IDs.
- **Configuration choices:**
  - mode: combine
  - advanced merge by fields:
    - invoice field `IdentyfikatorKontrahenta`
    - contractor field `Identyfikator`
- **Key expressions / variables:** merge keys above
- **Input / output connections:**
  - Input 0 from **Assign Endpoint to Invoice Type**
  - Input 1 from **Extract Contractor Data**
  - Outputs to:
    - **Invoices for Reminder**
    - **Y Days After Due Date**
    - **Z Days After Due Date**
- **Version-specific requirements:** Type version 3.2.
- **Edge cases / failures:**
  - Missing contractor records lead to unmatched invoices.
  - Multiple contractor records with same ID could duplicate invoices.
- **Sub-workflow reference:** None.

---

## Block 5 — Escalation Routing

### Overview
This block separates enriched invoices into three timed escalation paths: standard reminder, pre-trial notice, and court-stage notice.

### Nodes Involved
- Invoices for Reminder
- Payment Reminder
- Y Days After Due Date
- Pre-trial summon Reminder
- Z Days After Due Date
- Summons to court Reminder

### Node Details

#### 20. Invoices for Reminder
- **Type / role:** `Filter`; selects invoices for the reminder path.
- **Configuration choices:** OR logic:
  1. `TerminPlatnosci` equals today
  2. `TerminPlatnosci` equals today minus configured X-day value
- **Key expressions / variables:**
  - `{{ $json.TerminPlatnosci }}`
  - `{{ $now.format('yyyy-LL-dd') }}`
  - `{{ $now.minus($('Configuration').first().json["X days before due date"], 'days').format('yyyy-LL-dd') }}`
- **Input / output connections:** Input from **Merge Invoice with contractor** → output to **Payment Reminder**.
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:**
  - Same likely semantic issue as earlier: “X days before due date” is implemented as due date = today minus X.
  - If due dates include time components or different formats, equality may fail.
- **Sub-workflow reference:** None.

#### 21. Payment Reminder
- **Type / role:** `Set`; populates fields required by iFirma reminder sending.
- **Configuration choices:** Adds:
  - `Reminder Text`
  - payment method flags
  - sender mailbox
  - email template
  - `Send by Traditional Mail`
  - `Send E-Invoice`
  - includes original fields
- **Key expressions / variables:**
  - reminder body includes `PelnyNumer`, `DataWystawienia`, `Brutto`
- **Input / output connections:** Input from **Invoices for Reminder** → output to **URL for Sending Reminder**.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:**
  - Blank sender mailbox or template may be accepted or rejected depending on iFirma API rules.
  - If `endpoint` is missing, later request URL breaks.
- **Sub-workflow reference:** None.

#### 22. Y Days After Due Date
- **Type / role:** `Filter`; selects invoices exactly Y days after due date.
- **Configuration choices:** Compares formatted due date against `today - Y days`.
- **Key expressions / variables:**
  - `{{ new Date($json.TerminPlatnosci).format('yyyy-LL-dd') }}`
  - `{{ $now.minus($('Configuration').first().json["Y days after due date"], 'days').format('yyyy-LL-dd') }}`
- **Input / output connections:** Input from **Merge Invoice with contractor** → output to **Pre-trial summon Reminder**.
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:** Date format assumptions; strict comparison may miss malformed values.
- **Sub-workflow reference:** None.

#### 23. Pre-trial summon Reminder
- **Type / role:** `Set`; marks delivery flags for the pre-trial legal notice.
- **Configuration choices:**
  - `Send by Traditional Mail` = false
  - `Send by Email` = true
  - includes original fields
- **Key expressions / variables:** these booleans drive downstream If nodes.
- **Input / output connections:** Input from **Y Days After Due Date** → outputs to:
  - **Prepare Mail and Letter HTML Contents for Pre-trial summon**
  - **Pre-trial summon Slack Notification**
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:** None significant beyond delivery toggles.
- **Sub-workflow reference:** None.

#### 24. Z Days After Due Date
- **Type / role:** `Filter`; selects invoices exactly Z days after due date.
- **Configuration choices:** Compares formatted due date against `today - Z days`.
- **Key expressions / variables:** same pattern as Y filter.
- **Input / output connections:** Input from **Merge Invoice with contractor** → output to **Summons to court Reminder**.
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:** Same date-format issues as other filters.
- **Sub-workflow reference:** None.

#### 25. Summons to court Reminder
- **Type / role:** `Set`; marks delivery flags for the court-stage message.
- **Configuration choices:**
  - `Send by Traditional Mail` = false
  - `Send by Email` = true
  - includes original fields
- **Key expressions / variables:** delivery booleans used downstream.
- **Input / output connections:** Input from **Z Days After Due Date** → outputs to:
  - **Entering legal route Slack Notification**
  - **Prepare Mail and Letter HTML Contents for entering legal route**
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:** None significant beyond delivery flags.
- **Sub-workflow reference:** None.

---

## Block 6 — Payment Reminder via iFirma and Slack

### Overview
This branch sends a reminder through iFirma’s send endpoint and emits a Slack alert for internal visibility.

### Nodes Involved
- URL for Sending Reminder
- Payment Reminder Slack Notification
- Encode API Key for Sending Reminder
- Send Reminder

### Node Details

#### 26. URL for Sending Reminder
- **Type / role:** `Set`; constructs the iFirma send-reminder endpoint for the invoice.
- **Configuration choices:**
  - `url` = `https://www.ifirma.pl/iapi/{{$input.item.json.endpoint}}/send/{{$input.item.json.FakturaId}}.json`
  - `requestBody` = empty object
  - includes other fields
- **Key expressions / variables:** `endpoint`, `FakturaId`
- **Input / output connections:** Input from **Payment Reminder** → outputs to:
  - **Encode API Key for Sending Reminder**
  - **Payment Reminder Slack Notification**
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:**
  - Missing `endpoint` or `FakturaId` breaks the URL.
- **Sub-workflow reference:** None.

#### 27. Payment Reminder Slack Notification
- **Type / role:** `Slack`; sends a channel message summarizing the reminder event.
- **Configuration choices:** Uses Slack block message format with fields such as invoice number, overdue days, contractor, deadline, amount, issue date.
- **Key expressions / variables:**
  - `$json.PelnyNumer`
  - `$now.diffTo($json.TerminPlatnosci.toDateTime(), 'days').floor()`
  - `$json.KontrahentNazwa`
- **Input / output connections:** Input from **URL for Sending Reminder**.
- **Version-specific requirements:** Type version 2.4, Slack OAuth2 credential required.
- **Edge cases / failures:**
  - Channel ID must exist and app must have permission.
  - `TerminPlatnosci.toDateTime()` depends on n8n date coercion support.
- **Sub-workflow reference:** None.

#### 28. Encode API Key for Sending Reminder
- **Type / role:** `Code`; signs the send-reminder request.
- **Configuration choices:** Similar HMAC-SHA1 logic to previous auth nodes, but returns each incoming item enriched with `authHeader`.
- **Key expressions / variables:** same credential references as other auth nodes.
- **Input / output connections:** Input from **URL for Sending Reminder** → output to **Send Reminder**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - Same HMAC fragility as other auth nodes.
  - Contains unreachable code after first return.
  - Contains debug `console.log`.
- **Sub-workflow reference:** None.

#### 29. Send Reminder
- **Type / role:** `HTTP Request`; sends the payment reminder via iFirma.
- **Configuration choices:**
  - POST to dynamic send URL
  - body fields:
    - `Tekst`
    - `Przelew`
    - `Pobranie`
    - `MTransfer`
    - `SkrzynkaEmail `
    - `SzablonEmail `
    - `SkrzynkaEmailOdbiorcy `
  - query fields:
    - `wyslijEfaktura`
    - `wyslijPoczta`
  - `Authentication` header
- **Key expressions / variables:** fields from **Payment Reminder** and `authHeader`.
- **Input / output connections:** Input from **Encode API Key for Sending Reminder**.
- **Version-specific requirements:** Type version 4.3.
- **Edge cases / failures:**
  - Several body parameter names contain trailing spaces; this may be intentional for a copied API spec or may cause request mismatch.
  - Missing recipient email `EmailDlaFaktury` can fail or cause silent no-send.
  - iFirma may reject unsupported endpoint/invoice combinations.
- **Sub-workflow reference:** None.

---

## Block 7 — Pre-trial Summon Delivery

### Overview
This branch handles invoices at Y days overdue. It sends a Slack notification, generates legal-style HTML content, and conditionally sends by Gmail and/or physical mail through PostGrid.

### Nodes Involved
- Pre-trial summon Slack Notification
- Prepare Mail and Letter HTML Contents for Pre-trial summon
- Send pre-trial summon with post office?
- Send pre-trial summon letter via PostGrid
- Send pre-trial summon via Email?
- Send pre-trial summon mail

### Node Details

#### 30. Pre-trial summon Slack Notification
- **Type / role:** `Slack`; internal notification for pre-trial escalation.
- **Configuration choices:** Block message summarizing invoice details and overdue age.
- **Key expressions / variables:** invoice metadata fields.
- **Input / output connections:** Input from **Pre-trial summon Reminder**.
- **Version-specific requirements:** Type version 2.4, Slack OAuth2 credential.
- **Edge cases / failures:** Same Slack permission/channel issues as above.
- **Sub-workflow reference:** None.

#### 31. Prepare Mail and Letter HTML Contents for Pre-trial summon
- **Type / role:** `Set`; generates full HTML for email and postal letter.
- **Configuration choices:** Produces `Message Content` containing a styled formal debt notice with:
  - invoice details
  - overdue age
  - legal consequences
  - deadline based on `Z - Y`
  - bank transfer details from **Your Company Details**
- **Key expressions / variables:**
  - contractor fields like `KontrahentNazwa`, `Ulica`, `Miejscowosc`
  - company fields from **Your Company Details**
  - threshold math from **Configuration**
- **Input / output connections:** Input from **Pre-trial summon Reminder** → outputs to:
  - **Send pre-trial summon via Email?**
  - **Send pre-trial summon with post office?**
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:**
  - Missing contractor or company fields make content incomplete.
  - HTML is large; edits can easily break expressions.
- **Sub-workflow reference:** None.

#### 32. Send pre-trial summon with post office?
- **Type / role:** `If`; decides whether to send postal mail.
- **Configuration choices:** checks `Send by Traditional Mail == true`
- **Key expressions / variables:** `$json["Send by Traditional Mail"]`
- **Input / output connections:** Input from **Prepare Mail and Letter HTML Contents for Pre-trial summon** → true branch to **Send pre-trial summon letter via PostGrid**.
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:** None significant.
- **Sub-workflow reference:** None.

#### 33. Send pre-trial summon letter via PostGrid
- **Type / role:** `HTTP Request`; creates a physical letter via PostGrid print-mail API.
- **Configuration choices:**
  - POST `https://api.postgrid.com/print-mail/v1/letters`
  - JSON body includes `to`, `from`, `html`, `description`, print options
  - Uses generic custom auth credential
- **Key expressions / variables:**
  - contractor address fields
  - sender details from **Your Company Details**
  - `Message Content`
- **Input / output connections:** Input from **Send pre-trial summon with post office?** true branch.
- **Version-specific requirements:** Type version 4.3; requires `httpCustomAuth` credential with header `x-api-key`.
- **Edge cases / failures:**
  - Missing postal fields or invalid country code can cause PostGrid validation errors.
  - Non-Latin characters or HTML formatting could affect rendering.
- **Sub-workflow reference:** None.

#### 34. Send pre-trial summon via Email?
- **Type / role:** `If`; decides whether to send the email version.
- **Configuration choices:** checks `Send by Email == true`
- **Key expressions / variables:** `$json["Send by Email"]`
- **Input / output connections:** Input from **Prepare Mail and Letter HTML Contents for Pre-trial summon** → true branch to **Send pre-trial summon mail**.
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:** None significant.
- **Sub-workflow reference:** None.

#### 35. Send pre-trial summon mail
- **Type / role:** `Gmail`; sends the HTML legal notice by email.
- **Configuration choices:**
  - To = `EmailDlaFaktury`
  - Subject = `Pre-trial summon for invoice {{ PelnyNumer }}`
  - Message = `Message Content`
- **Key expressions / variables:** recipient email, invoice number, generated HTML.
- **Input / output connections:** Input from **Send pre-trial summon via Email?** true branch.
- **Version-specific requirements:** Type version 2.2; requires Gmail OAuth2 credential.
- **Edge cases / failures:**
  - Missing or invalid `EmailDlaFaktury`
  - Gmail quota, OAuth expiry, or restricted sending permissions
  - Depending on Gmail node behavior, HTML may need explicit HTML-mode handling
- **Sub-workflow reference:** None.

---

## Block 8 — Court / Legal Route Delivery

### Overview
This branch handles invoices at Z days overdue. It mirrors the pre-trial branch with a more severe legal message and conditional delivery by email and/or physical mail.

### Nodes Involved
- Entering legal route Slack Notification
- Prepare Mail and Letter HTML Contents for entering legal route
- Send entering legal route with post office?
- Send entering legal route letter via PostGrid
- Send entering legal route via Email?
- Send entering legal route mail

### Node Details

#### 36. Entering legal route Slack Notification
- **Type / role:** `Slack`; internal notice for the final escalation stage.
- **Configuration choices:** Block message with invoice summary.
- **Key expressions / variables:** invoice metadata fields.
- **Input / output connections:** Input from **Summons to court Reminder**.
- **Version-specific requirements:** Type version 2.4.
- **Edge cases / failures:** Same Slack auth/channel concerns.
- **Sub-workflow reference:** None.

#### 37. Prepare Mail and Letter HTML Contents for entering legal route
- **Type / role:** `Set`; creates final-stage HTML notice.
- **Configuration choices:** Builds a more severe legal/litigation-oriented `Message Content` with:
  - case reference
  - principal amount
  - warning about lawsuit, asset attachment, legal costs
  - final settlement deadline
  - bank instructions
- **Key expressions / variables:**
  - company details from **Your Company Details**
  - invoice fields
  - threshold math using configuration
- **Input / output connections:** Input from **Summons to court Reminder** → outputs to:
  - **Send entering legal route via Email?**
  - **Send entering legal route with post office?**
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:**
  - Same HTML maintenance and missing-data risks as pre-trial HTML.
- **Sub-workflow reference:** None.

#### 38. Send entering legal route with post office?
- **Type / role:** `If`; controls physical mail sending.
- **Configuration choices:** checks `Send by Traditional Mail == true`
- **Key expressions / variables:** `$json["Send by Traditional Mail"]`
- **Input / output connections:** Input from **Prepare Mail and Letter HTML Contents for entering legal route** → true branch to **Send entering legal route letter via PostGrid**.
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:** None significant.
- **Sub-workflow reference:** None.

#### 39. Send entering legal route letter via PostGrid
- **Type / role:** `HTTP Request`; posts final-stage letter to PostGrid.
- **Configuration choices:** Same endpoint and structure as pre-trial letter, but different HTML body.
- **Key expressions / variables:** contractor address, sender address, `Message Content`.
- **Input / output connections:** Input from **Send entering legal route with post office?** true branch.
- **Version-specific requirements:** Type version 4.3; same custom auth requirement.
- **Edge cases / failures:** Same as other PostGrid node.
- **Sub-workflow reference:** None.

#### 40. Send entering legal route via Email?
- **Type / role:** `If`; controls email sending.
- **Configuration choices:** checks `Send by Email == true`
- **Key expressions / variables:** `$json["Send by Email"]`
- **Input / output connections:** Input from **Prepare Mail and Letter HTML Contents for entering legal route** → true branch to **Send entering legal route mail**.
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:** None significant.
- **Sub-workflow reference:** None.

#### 41. Send entering legal route mail
- **Type / role:** `Gmail`; sends the court-stage message.
- **Configuration choices:**
  - To = `EmailDlaFaktury`
  - Subject = `Summon to court over invoice {{ FakturaId }}`
  - Message = `Message Content`
- **Key expressions / variables:** email recipient, invoice ID, HTML body.
- **Input / output connections:** Input from **Send entering legal route via Email?** true branch.
- **Version-specific requirements:** Type version 2.2; Gmail OAuth2 credential.
- **Edge cases / failures:** Same Gmail risks as other Gmail node.
- **Sub-workflow reference:** None.

---

## Block 9 — Documentation / Visual Annotation Nodes

### Overview
These sticky notes provide operational guidance, setup instructions, and context for maintainers. They do not affect execution but are important for reconstruction and operator understanding.

### Nodes Involved
- Sticky Note8
- Sticky Note9
- Sticky Note10
- Sticky Note11
- Sticky Note12
- Sticky Note13
- Sticky Note14
- Sticky Note15
- Sticky Note16
- Sticky Note17
- Sticky Note
- Sticky Note1

### Node Details
All of these are `Sticky Note` nodes, version 1, with no runtime role or connections. Their content is captured in the summary table where relevant.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Daily Check | n8n-nodes-base.scheduleTrigger | Daily workflow trigger |  | Configuration | ## Fill out config 1. Email/Login - Email or login used to log into the iFirma service 2. API Key Invoice - API key that can be found at: Start > Data and Configuration > Extensions and Integrations > API 3. X days before due date - Number of days before payment deadline for reminder - How many days before payment deadline to send reminder (default 7) 4. Y days after due date - Number of days after payment deadline to send notification about intent to pursue legal action 5. Z days after due date - Number of days after payment deadline to send notification about pursuing legal action |
| Configuration | n8n-nodes-base.set | Stores iFirma credentials and escalation thresholds | Schedule Daily Check | Your Company Details | ## Fill out config 1. Email/Login - Email or login used to log into the iFirma service 2. API Key Invoice - API key that can be found at: Start > Data and Configuration > Extensions and Integrations > API 3. X days before due date - Number of days before payment deadline for reminder - How many days before payment deadline to send reminder (default 7) 4. Y days after due date - Number of days after payment deadline to send notification about intent to pursue legal action 5. Z days after due date - Number of days after payment deadline to send notification about pursuing legal action |
| Your Company Details | n8n-nodes-base.set | Stores sender identity, address, and bank details | Configuration | URL to Fetch Invoices | ## Your company details (Needed for PostGrid) This data is needed for PostGrid API to work. It will be used to send letters in your name for Pre-trial summons and Summons to court when invoice is not paid |
| URL to Fetch Invoices | n8n-nodes-base.set | Defines invoice list API URL and request body placeholder | Your Company Details | Encode API Key | ## Step 1: Fetching Invoice List Fetching the invoice list from the service requires creating an encryption key that is not available to generate using standard N8N tools because it uses SHA-1 algorithm. Because iFirma returns error in response body, we first check if response contains an error, if not we extract items from the body to loop over them. |
| Encode API Key | n8n-nodes-base.code | Generates iFirma HMAC-SHA1 auth header for invoice retrieval | URL to Fetch Invoices | Fetch Invoices from System | ## Step 1: Fetching Invoice List Fetching the invoice list from the service requires creating an encryption key that is not available to generate using standard N8N tools because it uses SHA-1 algorithm. Because iFirma returns error in response body, we first check if response contains an error, if not we extract items from the body to loop over them. |
| Fetch Invoices from System | n8n-nodes-base.httpRequest | Retrieves unpaid/overdue invoices from iFirma | Encode API Key | Contains Error? | Gets all issued (unpaid) invoices from ifirma.pl |
| Contains Error? | n8n-nodes-base.if | Detects application-level iFirma invoice fetch errors | Fetch Invoices from System | Error Fetching Invoices; Extract Invoices | ## Step 1: Fetching Invoice List Fetching the invoice list from the service requires creating an encryption key that is not available to generate using standard N8N tools because it uses SHA-1 algorithm. Because iFirma returns error in response body, we first check if response contains an error, if not we extract items from the body to loop over them. |
| Error Fetching Invoices | n8n-nodes-base.stopAndError | Stops execution when invoice fetch fails | Contains Error? |  | ## Step 1: Fetching Invoice List Fetching the invoice list from the service requires creating an encryption key that is not available to generate using standard N8N tools because it uses SHA-1 algorithm. Because iFirma returns error in response body, we first check if response contains an error, if not we extract items from the body to loop over them. |
| Extract Invoices | n8n-nodes-base.splitOut | Splits invoice array into one item per invoice | Contains Error? | Filter Unpaid Invoices Due Today or Earlier | ## Step 1: Fetching Invoice List Fetching the invoice list from the service requires creating an encryption key that is not available to generate using standard N8N tools because it uses SHA-1 algorithm. Because iFirma returns error in response body, we first check if response contains an error, if not we extract items from the body to loop over them. |
| Filter Unpaid Invoices Due Today or Earlier | n8n-nodes-base.filter | Keeps due/overdue and reminder-window invoices | Extract Invoices | Assign Endpoint to Invoice Type | ## Step 2: Filtering paid invoices, map appropriate endpoints Invoices are filtered based on the `TerminPlatnosci` (Payment Deadline) value. Based on `Rodzaj` value from the response we map appropriate endpoint to our object. |
| Assign Endpoint to Invoice Type | n8n-nodes-base.code | Maps invoice type to iFirma send endpoint | Filter Unpaid Invoices Due Today or Earlier | Deduplicate Contractors; Merge Invoice with contractor | ## Step 2: Filtering paid invoices, map appropriate endpoints Invoices are filtered based on the `TerminPlatnosci` (Payment Deadline) value. Based on `Rodzaj` value from the response we map appropriate endpoint to our object. |
| Deduplicate Contractors | n8n-nodes-base.removeDuplicates | Prevents duplicate contractor lookups | Assign Endpoint to Invoice Type | URL to Fetch Contractor Information | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| URL to Fetch Contractor Information | n8n-nodes-base.set | Builds contractor lookup endpoint | Deduplicate Contractors | Encode API Key to Fetch Contractor Information | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| Encode API Key to Fetch Contractor Information | n8n-nodes-base.code | Generates auth header for contractor lookup | URL to Fetch Contractor Information | Fetch Contractor Information | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| Fetch Contractor Information | n8n-nodes-base.httpRequest | Retrieves contractor details from iFirma | Encode API Key to Fetch Contractor Information | Contains Error? 2 | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| Contains Error? 2 | n8n-nodes-base.if | Detects application-level contractor fetch errors | Fetch Contractor Information | Error Fetching Contractor Data; Extract Contractor Data | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| Error Fetching Contractor Data | n8n-nodes-base.stopAndError | Stops execution when contractor fetch fails | Contains Error? 2 |  | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| Extract Contractor Data | n8n-nodes-base.set | Extracts contractor object from API response | Contains Error? 2 | Merge Invoice with contractor | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| Merge Invoice with contractor | n8n-nodes-base.merge | Joins invoice and contractor data by contractor ID | Assign Endpoint to Invoice Type; Extract Contractor Data | Invoices for Reminder; Y Days After Due Date; Z Days After Due Date | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| Invoices for Reminder | n8n-nodes-base.filter | Selects invoices for reminder path | Merge Invoice with contractor | Payment Reminder | ## Step 4: Logic Separation At this point, the logic separates invoices into: 1. Payment reminder 2. Notification about intent to pursue legal action 3. Notification about pursuing legal action |
| Payment Reminder | n8n-nodes-base.set | Adds iFirma reminder payload fields | Invoices for Reminder | URL for Sending Reminder | ## Step 4: Logic Separation At this point, the logic separates invoices into: 1. Payment reminder 2. Notification about intent to pursue legal action 3. Notification about pursuing legal action |
| Y Days After Due Date | n8n-nodes-base.filter | Selects invoices for pre-trial escalation | Merge Invoice with contractor | Pre-trial summon Reminder | ## Step 4: Logic Separation At this point, the logic separates invoices into: 1. Payment reminder 2. Notification about intent to pursue legal action 3. Notification about pursuing legal action |
| Pre-trial summon Reminder | n8n-nodes-base.set | Sets delivery options for pre-trial notice | Y Days After Due Date | Prepare Mail and Letter HTML Contents for Pre-trial summon; Pre-trial summon Slack Notification | ## Step 4: Logic Separation At this point, the logic separates invoices into: 1. Payment reminder 2. Notification about intent to pursue legal action 3. Notification about pursuing legal action |
| Z Days After Due Date | n8n-nodes-base.filter | Selects invoices for court/legal-route escalation | Merge Invoice with contractor | Summons to court Reminder | ## Step 4: Logic Separation At this point, the logic separates invoices into: 1. Payment reminder 2. Notification about intent to pursue legal action 3. Notification about pursuing legal action |
| Summons to court Reminder | n8n-nodes-base.set | Sets delivery options for final legal notice | Z Days After Due Date | Entering legal route Slack Notification; Prepare Mail and Letter HTML Contents for entering legal route | ## Step 4: Logic Separation At this point, the logic separates invoices into: 1. Payment reminder 2. Notification about intent to pursue legal action 3. Notification about pursuing legal action |
| URL for Sending Reminder | n8n-nodes-base.set | Builds dynamic iFirma reminder-send endpoint | Payment Reminder | Encode API Key for Sending Reminder; Payment Reminder Slack Notification | ## Configuration These are values that are going to be added to the requests sent to iFirma  Values you can fill in can be found [here](https://api.ifirma.pl/wysylanie-faktur-poczta-tradycyjna-oraz-na-adres-e-mail-kontrahenta/) |
| Payment Reminder Slack Notification | n8n-nodes-base.slack | Sends Slack alert for standard reminder | URL for Sending Reminder |  | ## Notify Contractor About Approaching Payment Deadline |
| Encode API Key for Sending Reminder | n8n-nodes-base.code | Generates auth header for iFirma reminder send API | URL for Sending Reminder | Send Reminder | ## Notify Contractor About Approaching Payment Deadline |
| Send Reminder | n8n-nodes-base.httpRequest | Sends reminder via iFirma send endpoint | Encode API Key for Sending Reminder |  | ## Notify Contractor About Approaching Payment Deadline |
| Pre-trial summon Slack Notification | n8n-nodes-base.slack | Sends Slack alert for pre-trial notice | Pre-trial summon Reminder |  | ## Notify contractor about will to enter legal route |
| Prepare Mail and Letter HTML Contents for Pre-trial summon | n8n-nodes-base.set | Generates pre-trial email/letter HTML | Pre-trial summon Reminder | Send pre-trial summon via Email?; Send pre-trial summon with post office? | ## Notify contractor about will to enter legal route |
| Send pre-trial summon with post office? | n8n-nodes-base.if | Checks whether to send physical mail | Prepare Mail and Letter HTML Contents for Pre-trial summon | Send pre-trial summon letter via PostGrid | ## Notify contractor about will to enter legal route |
| Send pre-trial summon letter via PostGrid | n8n-nodes-base.httpRequest | Sends pre-trial letter through PostGrid | Send pre-trial summon with post office? |  | ## Notify contractor about will to enter legal route |
| Send pre-trial summon via Email? | n8n-nodes-base.if | Checks whether to send email | Prepare Mail and Letter HTML Contents for Pre-trial summon | Send pre-trial summon mail | ## Notify contractor about will to enter legal route |
| Send pre-trial summon mail | n8n-nodes-base.gmail | Sends pre-trial email notice | Send pre-trial summon via Email? |  | ## Notify contractor about will to enter legal route |
| Entering legal route Slack Notification | n8n-nodes-base.slack | Sends Slack alert for final legal escalation | Summons to court Reminder |  | ## Notify contractor about entering legal route |
| Prepare Mail and Letter HTML Contents for entering legal route | n8n-nodes-base.set | Generates final legal email/letter HTML | Summons to court Reminder | Send entering legal route via Email?; Send entering legal route with post office? | ## Notify contractor about entering legal route |
| Send entering legal route with post office? | n8n-nodes-base.if | Checks whether to send final legal letter physically | Prepare Mail and Letter HTML Contents for entering legal route | Send entering legal route letter via PostGrid | ## Notify contractor about entering legal route |
| Send entering legal route letter via PostGrid | n8n-nodes-base.httpRequest | Sends final legal letter through PostGrid | Send entering legal route with post office? |  | ## Notify contractor about entering legal route |
| Send entering legal route via Email? | n8n-nodes-base.if | Checks whether to send final legal notice by email | Prepare Mail and Letter HTML Contents for entering legal route | Send entering legal route mail | ## Notify contractor about entering legal route |
| Send entering legal route mail | n8n-nodes-base.gmail | Sends final legal notice via Gmail | Send entering legal route via Email? |  | ## Notify contractor about entering legal route |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## iFirma Overdue Invoice Vindication  **This workflow automates payment recovery for overdue invoices issued in** [iFirma](https://ifirma.pl)**. It fetches unpaid invoices daily, enriches them with full contractor data, and sends the appropriate notification — payment reminder, pre-trial summons, or court summons — via Gmail and/or physical mail through** [PostGrid](https://postgrid.com)**. Every action is reported to a Slack channel.**  ### How it works Each day the workflow fetches all unpaid invoices from iFirma using HMAC-SHA1 authenticated API calls, then enriches each invoice with full contractor details. Invoices are split into three escalation tiers based on configurable day thresholds:  - **Reminder** — due today or X days before due date - **Pre-trial summons** — Y days after due date - **Court summons** — Z days after due date  Each tier can independently send email, a physical letter via PostGrid, or both.  ### Setup 1. Fill out the **Configuration** node with your iFirma login, API key (found under *Data and Configuration → Extensions and Integrations → API*), and the X/Y/Z day thresholds 2. Fill out **Your Company Details** — used in letter and email templates and as the PostGrid sender address 3. Configure PostGrid auth in both PostGrid nodes using Custom Auth: `{ "headers": { "x-api-key": "" } }`  ### How to customize The **Payment Reminder**, **Pre-trial summon Reminder**, and **Summons to court Reminder** nodes each have `Send by Traditional Mail` and `Send by Email` toggles — flip these to control delivery channel per escalation tier. HTML templates can be freely edited in the corresponding **Prepare Mail and Letter HTML Contents** nodes.  Need help? Contact us at [developers@sailingbyte.com](mailto:developers@sailingbyte.com) or visit [sailingbyte.com](https://sailingbyte.com)!  Happy hacking! |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Fill out config 1. Email/Login - Email or login used to log into the iFirma service 2. API Key Invoice - API key that can be found at: Start > Data and Configuration > Extensions and Integrations > API 3. X days before due date - Number of days before payment deadline for reminder - How many days before payment deadline to send reminder (default 7) 4. Y days after due date - Number of days after payment deadline to send notification about intent to pursue legal action 5. Z days after due date - Number of days after payment deadline to send notification about pursuing legal action |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Step 1: Fetching Invoice List Fetching the invoice list from the service requires creating an encryption key that is not available to generate using standard N8N tools because it uses SHA-1 algorithm. Because iFirma returns error in response body, we first check if response contains an error, if not we extract items from the body to loop over them. |
| Sticky Note11 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Step 2: Filtering paid invoices, map appropriate endpoints Invoices are filtered based on the `TerminPlatnosci` (Payment Deadline) value. Based on `Rodzaj` value from the response we map appropriate endpoint to our object. |
| Sticky Note12 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Configuration These are values that are going to be added to the requests sent to iFirma  Values you can fill in can be found [here](https://api.ifirma.pl/wysylanie-faktur-poczta-tradycyjna-oraz-na-adres-e-mail-kontrahenta/) |
| Sticky Note13 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Step 3: Fetch Contractor Data and Merge with Invoice Data Because iFirma sends only partial data about contractor we need to make additional fetches so we have complete object with all necessary data. For these we use `IdentyfikatorKontrahenta` |
| Sticky Note14 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Step 4: Logic Separation At this point, the logic separates invoices into: 1. Payment reminder 2. Notification about intent to pursue legal action 3. Notification about pursuing legal action |
| Sticky Note15 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Notify Contractor About Approaching Payment Deadline |
| Sticky Note16 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Your company details (Needed for PostGrid) This data is needed for PostGrid API to work. It will be used to send letters in your name for Pre-trial summons and Summons to court when invoice is not paid |
| Sticky Note17 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Notify contractor about will to enter legal route |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Notify contractor about entering legal route |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Step 5: Send reminder |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**
   - Name: `Schedule Daily Check`
   - Configure it to run once every 24 hours.
   - If you need an exact local time such as 9:00 AM, prefer a cron-style schedule instead of a simple hourly interval.

3. **Add a Set node**
   - Name: `Configuration`
   - Add these fields:
     - `Email/Login` as string
     - `API Key Invoice` as string
     - `X days before due date` as number, default `7`
     - `Y days after due date` as number, default `7`
     - `Z days after due date` as number, default `14`

4. **Connect**
   - `Schedule Daily Check` → `Configuration`

5. **Add another Set node**
   - Name: `Your Company Details`
   - Add:
     - `Company Name`
     - `Email`
     - `Phone`
     - `Country Code`
     - `City`
     - `Street`
     - `Postal Code`
     - `Swift Code`
     - `Bank Name`
     - `Bank Account Number`
     - `TIN (Tax Identification Number)`

6. **Connect**
   - `Configuration` → `Your Company Details`

7. **Add a Set node**
   - Name: `URL to Fetch Invoices`
   - Fields:
     - `url` = `https://www.ifirma.pl/iapi/faktury.json`
     - `requestBody` = empty string

8. **Connect**
   - `Your Company Details` → `URL to Fetch Invoices`

9. **Add a Code node**
   - Name: `Encode API Key`
   - Paste the custom HMAC-SHA1 JavaScript used to generate the iFirma `Authentication` header.
   - Ensure it:
     - reads `API Key Invoice` and `Email/Login` from `Configuration`
     - reads `url` and `requestBody` from input
     - builds `IAPIS user=<login>, hmac-sha1=<hash>`
   - Output should include `authHeader`

10. **Connect**
    - `URL to Fetch Invoices` → `Encode API Key`

11. **Add an HTTP Request node**
    - Name: `Fetch Invoices from System`
    - Method: `GET`
    - URL: `{{ $('URL to Fetch Invoices').first().json.url }}`
    - Send headers: enabled
    - Header:
      - `Authentication` = `{{ $json.authHeader }}`
    - Send query parameters: enabled
    - Query parameter:
      - `status` = `przeterminowane,nieoplacone,oplaconeCzesciowo`
    - If you need multipage retrieval, configure pagination carefully for iFirma’s page parameter `strona`

12. **Connect**
    - `Encode API Key` → `Fetch Invoices from System`

13. **Add an If node**
    - Name: `Contains Error?`
    - Condition:
      - `{{ $json.response.Kod }}` greater than `0`

14. **Connect**
    - `Fetch Invoices from System` → `Contains Error?`

15. **Add a Stop And Error node**
    - Name: `Error Fetching Invoices`
    - Use an error object such as:
      - message: `Error fetching invoices`
      - body: serialized invoice API response

16. **Connect**
    - `Contains Error?` true → `Error Fetching Invoices`

17. **Add a Split Out node**
    - Name: `Extract Invoices`
    - Field to split out: `response.Wynik`

18. **Connect**
    - `Contains Error?` false → `Extract Invoices`

19. **Add a Filter node**
    - Name: `Filter Unpaid Invoices Due Today or Earlier`
    - OR conditions:
      - `new Date($json.TerminPlatnosci)` is before or equal to `$now`
      - or due date equals the configured reminder comparison date
    - Important: review the intended reminder semantics. If you really want “X days before due date”, compare due date to `$now.plus(X, 'days')`, not `$now.minus(...)`.

20. **Connect**
    - `Extract Invoices` → `Filter Unpaid Invoices Due Today or Earlier`

21. **Add a Code node**
    - Name: `Assign Endpoint to Invoice Type`
    - Create a JavaScript mapping from `Rodzaj` to endpoint, for example:
      - `prz_faktura_kraj` → `fakturakraj`
      - `prz_faktura_wys_ter_kraj` → `fakturawaluta`
      - `prz_faktura_proforma` → `fakturaproformakraj`
      - `prz_faktura_zal` → `fakturazaliczka`
      - `prz_faktura_kon` → `fakturakoncowa`
      - `prz_rachunek_kraj` → `rachunekkraj`
      - `prz_rachunek_zagr` → `rachunekue`
    - For each incoming item, add field `endpoint`

22. **Connect**
    - `Filter Unpaid Invoices Due Today or Earlier` → `Assign Endpoint to Invoice Type`

23. **Add a Remove Duplicates node**
    - Name: `Deduplicate Contractors`
    - Compare selected fields
    - Field: `IdentyfikatorKontrahenta`

24. **Connect**
    - `Assign Endpoint to Invoice Type` → `Deduplicate Contractors`

25. **Add a Set node**
    - Name: `URL to Fetch Contractor Information`
    - Fields:
      - `url` = `https://www.ifirma.pl/iapi/kontrahenci/id/{{$json.IdentyfikatorKontrahenta}}.json`
      - `requestBody` = empty string

26. **Connect**
    - `Deduplicate Contractors` → `URL to Fetch Contractor Information`

27. **Add a Code node**
    - Name: `Encode API Key to Fetch Contractor Information`
    - Reuse the same HMAC-SHA1 logic as before
    - Output `authHeader`

28. **Connect**
    - `URL to Fetch Contractor Information` → `Encode API Key to Fetch Contractor Information`

29. **Add an HTTP Request node**
    - Name: `Fetch Contractor Information`
    - Method: `GET`
    - URL should be: `{{ $('URL to Fetch Contractor Information').item.json.url }}`
    - Do not use the whole JSON object as URL.
    - Header:
      - `Authentication` = `{{ $json.authHeader }}`

30. **Connect**
    - `Encode API Key to Fetch Contractor Information` → `Fetch Contractor Information`

31. **Add an If node**
    - Name: `Contains Error? 2`
    - Condition:
      - `{{ $json.response.Kod }}` greater than `0`

32. **Connect**
    - `Fetch Contractor Information` → `Contains Error? 2`

33. **Add a Stop And Error node**
    - Name: `Error Fetching Contractor Data`
    - Recommended error body should use the contractor fetch response, not the invoice fetch response.

34. **Connect**
    - `Contains Error? 2` true → `Error Fetching Contractor Data`

35. **Add a Set node**
    - Name: `Extract Contractor Data`
    - Use raw JSON output mode
    - Output: `{{ JSON.stringify($input.item.json.response.Wynik[0], null, 2) }}`

36. **Connect**
    - `Contains Error? 2` false → `Extract Contractor Data`

37. **Add a Merge node**
    - Name: `Merge Invoice with contractor`
    - Mode: combine
    - Merge by fields:
      - input 1 field `IdentyfikatorKontrahenta`
      - input 2 field `Identyfikator`

38. **Connect**
    - `Assign Endpoint to Invoice Type` → `Merge Invoice with contractor` input 1
    - `Extract Contractor Data` → `Merge Invoice with contractor` input 2

39. **Add a Filter node**
    - Name: `Invoices for Reminder`
    - OR conditions:
      - due date equals today
      - or due date equals the reminder comparison date
    - Again, validate whether you want plus-X or minus-X logic.

40. **Add a Set node**
    - Name: `Payment Reminder`
    - Enable “include other fields”
    - Add:
      - `Reminder Text`
      - `Can Pay by Transfer` = true
      - `Can Pay Cash on Delivery` = false
      - `Can Pay MTransfer` = false
      - `Email Address to Send From`
      - `Email Template`
      - `Send by Traditional Mail` = false
      - `Send E-Invoice` = true

41. **Connect**
    - `Merge Invoice with contractor` → `Invoices for Reminder`
    - `Invoices for Reminder` → `Payment Reminder`

42. **Add a Filter node**
    - Name: `Y Days After Due Date`
    - Condition: due date equals `today - Y days`

43. **Add a Set node**
    - Name: `Pre-trial summon Reminder`
    - Include other fields
    - Add:
      - `Send by Traditional Mail` = false
      - `Send by Email` = true

44. **Connect**
    - `Merge Invoice with contractor` → `Y Days After Due Date`
    - `Y Days After Due Date` → `Pre-trial summon Reminder`

45. **Add a Filter node**
    - Name: `Z Days After Due Date`
    - Condition: due date equals `today - Z days`

46. **Add a Set node**
    - Name: `Summons to court Reminder`
    - Include other fields
    - Add:
      - `Send by Traditional Mail` = false
      - `Send by Email` = true

47. **Connect**
    - `Merge Invoice with contractor` → `Z Days After Due Date`
    - `Z Days After Due Date` → `Summons to court Reminder`

48. **Add a Set node**
    - Name: `URL for Sending Reminder`
    - Include other fields
    - Fields:
      - `url` = `https://www.ifirma.pl/iapi/{{$input.item.json.endpoint}}/send/{{$input.item.json.FakturaId}}.json`
      - `requestBody` = empty object

49. **Connect**
    - `Payment Reminder` → `URL for Sending Reminder`

50. **Add a Slack node**
    - Name: `Payment Reminder Slack Notification`
    - Authentication: Slack OAuth2
    - Select target channel
    - Use block message mode
    - Add invoice summary fields

51. **Connect**
    - `URL for Sending Reminder` → `Payment Reminder Slack Notification`

52. **Add a Code node**
    - Name: `Encode API Key for Sending Reminder`
    - Reuse the HMAC-SHA1 signing logic
    - Return all incoming items enriched with `authHeader`

53. **Connect**
    - `URL for Sending Reminder` → `Encode API Key for Sending Reminder`

54. **Add an HTTP Request node**
    - Name: `Send Reminder`
    - Method: `POST`
    - URL: `{{ $('URL for Sending Reminder').first().json.url }}`
    - Headers:
      - `Authentication` = `{{ $json.authHeader }}`
    - Query:
      - `wyslijEfaktura` = `{{ $json['Send E-Invoice'] }}`
      - `wyslijPoczta` = `{{ $json['Send by Traditional Mail'] }}`
    - Body fields:
      - `Tekst`
      - `Przelew`
      - `Pobranie`
      - `MTransfer`
      - `SkrzynkaEmail`
      - `SzablonEmail`
      - `SkrzynkaEmailOdbiorcy`
    - Remove any trailing spaces from field names unless iFirma explicitly requires them.

55. **Connect**
    - `Encode API Key for Sending Reminder` → `Send Reminder`

56. **Add a Slack node**
    - Name: `Pre-trial summon Slack Notification`
    - Slack OAuth2
    - Channel of your choice
    - Block message with invoice details

57. **Connect**
    - `Pre-trial summon Reminder` → `Pre-trial summon Slack Notification`

58. **Add a Set node**
    - Name: `Prepare Mail and Letter HTML Contents for Pre-trial summon`
    - Include other fields
    - Add `Message Content` as a large HTML string
    - Reference:
      - contractor fields
      - company details
      - invoice details
      - `Y` and `Z` thresholds
    - This content is used for both Gmail and PostGrid

59. **Connect**
    - `Pre-trial summon Reminder` → `Prepare Mail and Letter HTML Contents for Pre-trial summon`

60. **Add an If node**
    - Name: `Send pre-trial summon with post office?`
    - Condition:
      - `{{ $json["Send by Traditional Mail"] }}` equals `true`

61. **Add an If node**
    - Name: `Send pre-trial summon via Email?`
    - Condition:
      - `{{ $json["Send by Email"] }}` equals `true`

62. **Connect**
    - `Prepare Mail and Letter HTML Contents for Pre-trial summon` → both If nodes

63. **Add an HTTP Request node**
    - Name: `Send pre-trial summon letter via PostGrid`
    - Method: `POST`
    - URL: `https://api.postgrid.com/print-mail/v1/letters`
    - Authentication: Generic Credential Type → HTTP Custom Auth
    - Credential setup:
      - custom header `x-api-key: <your_postgrid_api_key>`
    - JSON body:
      - `to` from contractor address fields
      - `from` from `Your Company Details`
      - `html` from `Message Content`
      - print options like color, double sided, address placement

64. **Connect**
    - `Send pre-trial summon with post office?` true → `Send pre-trial summon letter via PostGrid`

65. **Add a Gmail node**
    - Name: `Send pre-trial summon mail`
    - Credential: Gmail OAuth2
    - To: `{{ $json.EmailDlaFaktury }}`
    - Subject: `Pre-trial summon for invoice {{ $json.PelnyNumer }}`
    - Message: `{{ $json["Message Content"] }}`

66. **Connect**
    - `Send pre-trial summon via Email?` true → `Send pre-trial summon mail`

67. **Add a Slack node**
    - Name: `Entering legal route Slack Notification`
    - Slack OAuth2
    - Configure similar invoice summary block

68. **Connect**
    - `Summons to court Reminder` → `Entering legal route Slack Notification`

69. **Add a Set node**
    - Name: `Prepare Mail and Letter HTML Contents for entering legal route`
    - Include other fields
    - Add `Message Content` as final-stage legal HTML

70. **Connect**
    - `Summons to court Reminder` → `Prepare Mail and Letter HTML Contents for entering legal route`

71. **Add an If node**
    - Name: `Send entering legal route with post office?`
    - Condition:
      - `{{ $json["Send by Traditional Mail"] }}` equals `true`

72. **Add an If node**
    - Name: `Send entering legal route via Email?`
    - Condition:
      - `{{ $json["Send by Email"] }}` equals `true`

73. **Connect**
    - `Prepare Mail and Letter HTML Contents for entering legal route` → both If nodes

74. **Add an HTTP Request node**
    - Name: `Send entering legal route letter via PostGrid`
    - Same configuration as the pre-trial PostGrid node, but with the legal-route HTML

75. **Connect**
    - `Send entering legal route with post office?` true → `Send entering legal route letter via PostGrid`

76. **Add a Gmail node**
    - Name: `Send entering legal route mail`
    - Credential: Gmail OAuth2
    - To: `{{ $json.EmailDlaFaktury }}`
    - Subject: `Summon to court over invoice {{ $json.FakturaId }}`
    - Message: `{{ $json["Message Content"] }}`

77. **Connect**
    - `Send entering legal route via Email?` true → `Send entering legal route mail`

78. **Configure credentials**
    - **Slack OAuth2** for all Slack nodes
    - **Gmail OAuth2** for both Gmail nodes
    - **HTTP Custom Auth** for both PostGrid nodes with:
      - header name: `x-api-key`
      - header value: your PostGrid API key

79. **Fill business values**
    - In `Configuration`, enter iFirma login and API key
    - In `Your Company Details`, enter valid sender and banking details
    - In reminder/legal Set nodes, adjust delivery toggles as needed

80. **Recommended corrections before production**
    - Fix `Fetch Contractor Information` URL expression to reference `.url`
    - Fix `Error Fetching Contractor Data` to serialize contractor response
    - Review all “X days before due date” logic
    - Validate `Send Reminder` parameter names for trailing spaces
    - Consider adding explicit error handling after Gmail/PostGrid/iFirma send nodes
    - Consider converting repeated HMAC code into a shared sub-workflow or reusable custom node if your environment supports it

**Sub-workflow setup:**  
This workflow contains **no Execute Workflow / sub-workflow nodes**. No separate child workflow is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| iFirma overdue invoice vindication workflow: fetches unpaid invoices daily, enriches with contractor data, and sends reminders / pre-trial summons / court summons via Gmail and/or PostGrid, with Slack reporting. | [iFirma](https://ifirma.pl), [PostGrid](https://postgrid.com) |
| iFirma API key location: Data and Configuration → Extensions and Integrations → API | iFirma account settings |
| PostGrid custom auth example: `{ "headers": { "x-api-key": "" } }` | PostGrid credential setup |
| iFirma values for reminder sending can be found in the API documentation. | https://api.ifirma.pl/wysylanie-faktur-poczta-tradycyjna-oraz-na-adres-e-mail-kontrahenta/ |
| Contact for help: developers@sailingbyte.com | mailto:developers@sailingbyte.com |
| Project/company link | https://sailingbyte.com |

## Additional implementation observations
- The workflow description mentions Gmail, PostGrid, and Slack broadly, but only the **pre-trial** and **court** branches use Gmail/PostGrid. The **payment reminder** branch uses iFirma and Slack.
- Several nodes contain duplicated custom SHA-1/HMAC code. This works but increases maintenance risk.
- There are at least three likely implementation issues worth reviewing before deployment:
  1. contractor fetch URL expression appears incorrect
  2. contractor error node references the wrong upstream response
  3. reminder lead-time logic appears inverted relative to its label