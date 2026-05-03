Manage supplier quote requests and follow-ups with Gmail, Sheets, GPT-4o-mini and WhatsApp

https://n8nworkflows.xyz/workflows/manage-supplier-quote-requests-and-follow-ups-with-gmail--sheets--gpt-4o-mini-and-whatsapp-12638


# Manage supplier quote requests and follow-ups with Gmail, Sheets, GPT-4o-mini and WhatsApp

## 1. Workflow Overview

**Title (given):** Manage supplier quote requests and follow-ups with Gmail, Sheets, GPT-4o-mini and WhatsApp  
**Workflow name (in JSON):** Request and manage supplier quotes with automation

**Purpose:** Automates a procurement quote cycle end-to-end: sending quote requests to suppliers, monitoring Gmail for replies, extracting pricing data from attached PDFs using an AI extractor, writing results to a “Price Comparison” Google Sheet, and sending reminders via WhatsApp (Twilio) or email.

**Target use cases:**
- Small procurement teams requesting quotes from multiple suppliers and comparing pricing
- Reducing manual inbox monitoring and spreadsheet updates
- Standardizing follow-ups and ensuring suppliers respond within defined timelines

### Logical blocks (based on node dependencies)

**1.1 Module 1 — Quote Request Sender (manual trigger)**
- Generates supplier contact data, emails each supplier a quote request, and logs the request in a Supplier sheet.

**1.2 Module 2 — Response Monitoring (hourly)**
- Searches Gmail for quote responses, parses message metadata, and updates the Supplier sheet when a quote (attachment) is detected.

**1.3 Module 3 — AI Price Extraction (scheduled)**
- Reads suppliers marked “quote_received = Yes”, retrieves relevant emails with attachments, downloads PDFs, extracts text, uses an AI extraction node to produce structured data, then saves product rows to a Price Comparison sheet and stores supplier phone numbers back to the Supplier sheet.

**1.4 Module 4 — WhatsApp Follow-ups (scheduled)**
- Finds suppliers who haven’t replied after 3/5/7 days, prepares a personalized reminder, and sends via WhatsApp if a phone number exists, otherwise sends via Gmail; then increments follow-up tracking fields in the Supplier sheet.

---

## 2. Block-by-Block Analysis

### 2.1 Module 1 — Quote Request Sender

**Overview:** Creates a list of suppliers (currently hardcoded), sends a templated quote request email via Gmail, and appends supplier/request details into the Supplier Google Sheet for tracking.

**Nodes involved:**
- Execute workflow
- Hardcode suppliers details
- Reach out to suppliers
- Log to suppliers sheet
- Sticky Note (hardcoded supplier note)
- Sticky Note7 (module note)

#### Node: **Execute workflow**
- **Type / role:** Manual Trigger — starts Module 1 on demand.
- **Config:** No parameters.
- **Outputs to:** Hardcode suppliers details.
- **Edge cases:** None (manual execution only).

#### Node: **Hardcode suppliers details**
- **Type / role:** Code — emits one item per supplier.
- **Config choices:** JavaScript returns:
  - `name`, `email`, `category`
- **Key variables:** `return suppliers.map(s => ({ json: s }));`
- **Outputs to:** Reach out to suppliers.
- **Edge cases:**
  - Duplicate emails will cause multiple rows and emails.
  - Emails currently set to `user@example.com` (placeholder) and must be replaced.
  - This module does not read suppliers from Sheets; it only writes.

#### Node: **Reach out to suppliers**
- **Type / role:** Gmail (Send) — sends the quote request.
- **Config choices:**
  - **To:** `{{ $json.email }}`
  - **Subject:** `Price Quote Request - {{ $json.category }}`
  - **HTML body:** templated message, includes due date `{{ $now.plus(7, 'days').toFormat('yyyy-MM-dd') }}`
  - Attribution disabled.
- **Outputs to:** Log to suppliers sheet.
- **Credentials:** Gmail OAuth2.
- **Version:** 2.1
- **Edge cases / failures:**
  - Gmail auth/consent errors.
  - Sending limits / rate limits.
  - Invalid recipient addresses.

#### Node: **Log to suppliers sheet**
- **Type / role:** Google Sheets (Append) — records request tracking row.
- **Config choices:**
  - Writes: `supplier_name`, `supplier_email`, `category`, `request_date` (today), `follow_up_needed` = “Yes”
  - Date format: `{{ $now.toFormat('yyyy-MM-dd') }}`
  - Targets document “Supplier_sheet” (documentId `1xrurRmmG-...`) and sheet gid=0 (named “Sheet1” in cached name).
- **Matching:** Append (no matching columns).
- **Outputs:** None (end of Module 1 path).
- **Credentials:** Google Sheets OAuth2.
- **Version:** 4.4
- **Edge cases / failures:**
  - Wrong spreadsheet ID/sheet gid.
  - Column/schema mismatch if sheet headers differ.
  - Duplicated supplier rows on repeated runs.

**Sticky notes applied (context):**
- Hardcoded supplier note: “Code in JavaScript hardcodes the details of the suppliers…”
- Sticky Note7: “Reach out to suppliers and log details…”

---

### 2.2 Module 2 — Response Monitoring (hourly)

**Overview:** Every hour, searches Gmail for recent supplier replies with attachments, fetches full email details, parses sender/recipient to infer supplier email, checks for attachment indicators, then updates the Supplier sheet to mark quote received.

**Nodes involved:**
- Trigger workflow every hour
- Search for quote replies
- Get email details
- Parse email data
- Has attachments?
- Update supplier sheet
- Sticky Note1 (module header)
- Sticky Note8 (explanation)
- Sticky Note10 (IF branch explanation)

#### Node: **Trigger workflow every hour**
- **Type / role:** Schedule Trigger — starts Module 2.
- **Config choices:** Interval “hours” (every 1 hour).
- **Outputs to:** Search for quote replies.
- **Edge cases:**
  - Timezone considerations are governed by n8n instance settings.

#### Node: **Search for quote replies**
- **Type / role:** Gmail (Get Many) — searches for matching messages.
- **Config choices:**
  - Query: `subject:(Price Quote Request OR Re: Price Quote Request) newer_than:7d has:attachment`
  - Label: `INBOX`
  - Operation: `getAll`
- **Outputs to:** Get email details.
- **Credentials:** Gmail OAuth2.
- **Edge cases / failures:**
  - Gmail query may miss replies where subject differs or attachments aren’t present.
  - Large inbox results could be high-volume; consider limiting or pagination behavior.

#### Node: **Get email details**
- **Type / role:** Gmail (Get) — fetches a single message by ID with full structure.
- **Config choices:**
  - `messageId: {{ $json.id }}`
  - `simple: false`
  - Attachments not downloaded here (`downloadAttachments: false`)
- **Outputs to:** Parse email data.
- **Credentials:** Gmail OAuth2.
- **Edge cases:**
  - Message ID not found if deleted between steps.

#### Node: **Parse email data**
- **Type / role:** Code — normalizes email fields and infers supplier identity.
- **Config choices / logic:**
  - `fromEmail` from `$json.from.value[0].address`
  - `toEmail` from `$json.to.value[0].address`
  - Supplier email heuristic: if `toEmail` includes `+supplier` then use `toEmail` else `fromEmail`
  - “Has attachments” heuristic: checks whether `$json.text` or `$json.html` contains the word “attached”
  - Outputs `messageId`, `supplierEmail`, `receivedDate`, `emailText`, `hasAttachments`
- **Outputs to:** Has attachments?
- **Edge cases:**
  - The attachment detection is not based on Gmail’s attachment metadata; it’s keyword-based and can produce false positives/negatives.
  - The `+supplier` alias logic assumes a specific addressing scheme that is not otherwise created by this workflow.

#### Node: **Has attachments?**
- **Type / role:** IF — gates updates to Supplier sheet.
- **Condition:** `{{ $json.hasAttachments }} == true`
- **True output to:** Update supplier sheet
- **False output:** Not connected (no action taken)
- **Edge cases:**
  - If detection fails, Supplier sheet will not be updated even when Gmail search already required `has:attachment`.

#### Node: **Update supplier sheet**
- **Type / role:** Google Sheets (Update) — marks quote received.
- **Config choices:**
  - Matching column: `supplier_email`
  - Sets:
    - `status = "Quote Received"`
    - `quote_received = "Yes"`
    - `follow_up_needed = "No"`
    - `supplier_email = {{ $json.supplierEmail }}`
- **Outputs:** None.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - If `supplier_email` does not match exactly any row, update will not apply.
  - Multiple rows with same email can update multiple matches (depending on node behavior).

---

### 2.3 Module 3 — AI Price Extraction (scheduled)

**Overview:** Periodically reads Supplier sheet rows with `quote_received = Yes`, finds relevant Gmail messages with PDF attachments, downloads attachments, extracts PDF text, sends combined text to an information extraction AI node, parses extracted “Products List” into individual product rows, appends them to a Price Comparison sheet, and writes extracted phone numbers back to Supplier sheet.

**Nodes involved:**
- Trigger workflow daily
- Get quotes to process
- Get supplier email
- Download all attachments
- Extract from file
- Prepare for openAI
- Information Extractor
- Extract key information from invoice (AI model connector)
- Parse model output
- Save to Price Sheet
- Update supplier sheet with phone number
- No Operation, do nothing1
- Sticky Note2 (module header)
- Sticky Note6 (OpenAI note)
- Sticky Note9 (explanation)
- Sticky Note12 (parsing + updates note)

#### Node: **Trigger workflow daily**
- **Type / role:** Schedule Trigger — starts Module 3.
- **Config issue:** Despite the name “daily”, it is configured as an **hourly** interval (`{"field":"hours"}`) in the JSON. You likely want “days” here.
- **Outputs to:** Get quotes to process.
- **Edge cases:** Misconfigured cadence can cause repeated re-processing.

#### Node: **Get quotes to process**
- **Type / role:** Google Sheets (Read with filter) — selects suppliers whose quote is received.
- **Config choices:**
  - Filter UI: `quote_received == "Yes"`
  - Reads from Supplier_sheet documentId `1xrurRmmG-...`, gid=0
- **Outputs to:** Get supplier email.
- **Edge cases:**
  - If you never reset `quote_received` after processing, the same suppliers may be reprocessed each run.

#### Node: **Get supplier email**
- **Type / role:** Gmail (Get Many) — searches for quote request emails with attachments.
- **Config choices:**
  - Query: `subject:"Price Quote Request" has:attachment newer_than:7d`
  - Limit: 10
  - Operation: getAll
- **Outputs to:** Download all attachments.
- **Important limitation:** This search does **not** filter by the specific supplier email from the sheet row, so it may download unrelated suppliers’ attachments.
- **Edge cases:**
  - If >10 messages match, some suppliers could be skipped.

#### Node: **Download all attachments**
- **Type / role:** Gmail (Get) — downloads attachments for each matching email.
- **Config choices:**
  - `messageId: {{ $json.id }}`
  - `simple: false`
  - `downloadAttachments: true`
- **Outputs to:** Extract from file.
- **Edge cases:**
  - Very large attachments may hit memory limits or Gmail limits.

#### Node: **Extract from file**
- **Type / role:** Extract From File — converts PDF binary to text.
- **Config choices:**
  - Operation: `pdf`
  - `binaryPropertyName: {{ Object.keys($binary)[0] }}` (uses the first binary attachment)
- **Outputs to:** Prepare for openAI.
- **Edge cases:**
  - If the email has multiple attachments, only the first binary key is used per item.
  - Non-PDF attachments will fail PDF extraction.

#### Node: **Prepare for openAI**
- **Type / role:** Code — aggregates extracted text and attaches supplier context.
- **Config choices / logic:**
  - Concatenates `item.json.text || item.json.data` across all incoming items with separators.
  - Pulls supplier context from `$('Get quotes to process').all()[0]` (first row only).
  - Outputs:
    - `supplier_name`, `supplier_email`, `category`, `document_text`
  - If no extracted text: outputs `{ error: 'No extracted text found' }`
- **Outputs to:** Information Extractor.
- **Major edge case:** If multiple suppliers are pending, this node uses only the **first supplier row** but may combine PDFs from multiple emails, mixing suppliers.

#### Node: **Information Extractor**
- **Type / role:** LangChain Information Extractor — structured extraction from `document_text`.
- **Config choices:**
  - Input text: `{{ $json.document_text }}`
  - System prompt: extraction-only behavior; may omit unknowns.
  - Attributes requested (not all are used downstream):
    - Invoice Number, Client Name, Client Email, Total Amount, Invoice Date, Due Date
    - Products List (required; JSON string)
    - supplier_phone (required)
- **Model connection:** Uses **Extract key information from invoice** via `ai_languageModel` connection.
- **Outputs to:** Parse model output.
- **Edge cases:**
  - If the model returns invalid JSON for “Products List”, downstream parsing fails.
  - “Required: true” fields may cause extraction to fail or return empty depending on node behavior.

#### Node: **Extract key information from invoice**
- **Type / role:** OpenAI Chat Model node — provides the LLM used by Information Extractor.
- **Config choices:** Uses OpenAI credentials; no explicit model shown in parameters.
- **Connections:** Feeds as `ai_languageModel` into Information Extractor.
- **Credentials:** OpenAI API.
- **Edge cases:**
  - Model availability/quotas.
  - Increased cost if processing large PDFs frequently.

#### Node: **Parse model output**
- **Type / role:** Code — converts extracted products into per-product rows.
- **Config choices / logic:**
  - Reads: `$input.first().json.output['Products List']` (string)
  - `JSON.parse` into `products`; on parse error emits `{ error: 'Failed to parse products' }`
  - Builds rows:
    - supplier fields from `$('Prepare for openAI').first().json`
    - `price: parseFloat(product.price) || 0`
    - `currency` default `NGN`, `quantity` default `1`
    - `supplier_phone` read from extractor output
    - `extracted_date = today`
  - `source_file` uses `supplierData.file_name` but `file_name` is never set earlier, so it becomes `"unknown"`.
- **Outputs to:** (two parallel outputs) Update supplier sheet with phone number, Save to Price Sheet.
- **Edge cases:**
  - Trailing comma in returned JSON object is present in code; Node.js generally tolerates it in object literals, but keep consistent.
  - If extractor returns a non-array “Products List”, parsing succeeds but mapping may fail.

#### Node: **Save to Price Sheet**
- **Type / role:** Google Sheets (Append) — appends one row per product to Price Comparison sheet.
- **Config choices:**
  - Document: `1xHbz1EZQ...` (Price Comparison Sheet)
  - Sheet gid=0 (cached “Sheet1”)
  - Writes: supplier_name, supplier_email, product_name, price, currency, quantity, notes, extracted_date, category, source_file
- **Outputs to:** No Operation, do nothing1
- **Edge cases:**
  - Column names in sheet must match schema.
  - Price/currency formats should be consistent for later analysis.

#### Node: **Update supplier sheet with phone number**
- **Type / role:** Google Sheets (Update) — stores extracted phone number.
- **Config choices:**
  - Matching column: `supplier_email`
  - Updates: `phone_number = {{ $json.supplier_phone }}`
- **Edge cases:**
  - If supplier_phone extraction is wrong, it overwrites a correct phone number.
  - If multiple product rows are created, this update is repeated per product (same phone).

#### Node: **No Operation, do nothing1**
- **Type / role:** NoOp — terminator for the Price Sheet write branch.
- **Config:** None.

**Sticky notes applied (context):**
- Sticky Note6: “Obtain your API from https://platform.openai.com and connect your credentials”
- Sticky Note9/12 describe the intended flow (sheet read → email fetch → attachment → extraction → sheet updates).

---

### 2.4 Module 4 — WhatsApp Follow-ups (scheduled)

**Overview:** Periodically reads all suppliers from the Supplier sheet, filters those overdue for follow-up (3/5/7 days without quote), prepares a reminder message, chooses WhatsApp (Twilio) if phone exists otherwise Gmail, then updates follow-up counters and timestamps in the Supplier sheet.

**Nodes involved:**
- Trigger workflow daily1
- Get all quotes
- Filter pending quotes
- Any follow-ups needed?
- Prepare message
- Has phone number?
- Send WhatsApp follow-up message
- Send follow-up mail
- Update follow up count in supplier sheet
- No Operation, do nothing
- Sticky Note3 (module header)
- Sticky Note5 (Twilio sandbox config)
- Sticky Note11 (filtering explanation)
- Sticky Note13 (delivery explanation)

#### Node: **Trigger workflow daily1**
- **Type / role:** Schedule Trigger — starts Module 4.
- **Config issue:** `{"interval":[{}]}` is effectively ambiguous/misconfigured. It is labeled “daily1” but does not clearly specify daily cadence.
- **Outputs to:** Get all quotes.

#### Node: **Get all quotes**
- **Type / role:** Google Sheets (Read) — loads all supplier rows.
- **Config choices:** Reads from Supplier_sheet documentId `1xrurRmmG-...` gid=0.
- **Outputs to:** Filter pending quotes.
- **Edge cases:** Large sheets may increase execution time.

#### Node: **Filter pending quotes**
- **Type / role:** Code — selects suppliers needing follow-up.
- **Logic:**
  - Excludes rows where `quote_received === 'Yes'`
  - Calculates `daysSince` from `request_date` (or `request_sent_date`)
  - Keeps suppliers where `daysSince` is exactly 3, 5, or 7
- **Outputs to:** Any follow-ups needed?
- **Edge cases:**
  - If `request_date` is missing/invalid, `new Date()` can yield `Invalid Date` and daysSince becomes `NaN`, causing supplier to be skipped silently.
  - Equality checks (=== 3/5/7) mean if the schedule doesn’t run that day, follow-up is missed.

#### Node: **Any follow-ups needed?**
- **Type / role:** IF — checks if there are any items.
- **Condition:** `{{ $json.length }} > 0`
- **Issue:** Each incoming item is a supplier row; it does not have a `length` property. This condition will likely evaluate incorrectly. A more reliable pattern is checking `{{$items().length}}` or using an aggregate node.
- **True output to:** Prepare message
- **Edge cases:** Could prevent follow-ups from sending at all.

#### Node: **Prepare message**
- **Type / role:** Code — creates personalized reminder text and normalizes phone numbers.
- **Logic:**
  - Computes `daysSince` again
  - Sets message variants for day 3/5/≥7
  - Adds category and original request date
  - Formats phone number by prefixing `+` if missing
  - Outputs: supplier_name, supplier_email, phone_number, message, days_since_request, follow_up_type, follow_up_count
- **Outputs to:** Has phone number?
- **Edge cases:**
  - If phone number contains spaces or non-E.164 formatting, Twilio may reject it.

#### Node: **Has phone number?**
- **Type / role:** IF — routing between WhatsApp and email.
- **Condition:** `phone_number` is not empty (string notEmpty).
- **True output to:** Send WhatsApp follow-up message
- **False output to:** Send follow-up mail

#### Node: **Send WhatsApp follow-up message**
- **Type / role:** Twilio — sends WhatsApp message (sandbox or production).
- **Config choices:**
  - `to: {{ $json.phone_number }}`
  - `from: +1234567890` (placeholder; must be your Twilio WhatsApp-enabled number)
  - `toWhatsapp: true`
  - `message: {{ $json.message }}`
- **Outputs to:** Update follow up count in supplier sheet
- **Credentials:** Twilio API
- **Edge cases / failures:**
  - Sandbox requires joining code; messages fail if recipient hasn’t joined.
  - “From” must be WhatsApp-enabled in Twilio.
  - Phone number must be valid E.164.

#### Node: **Send follow-up mail**
- **Type / role:** Gmail (Send) — fallback if no phone.
- **Config choices:**
  - To: `{{ $json.supplier_email }}`
  - Subject: `Follow-up: Price Quote Request - {{ $json.supplier_name }}`
  - Body: `{{ $json.message }}` as plain text
- **Outputs to:** Update follow up count in supplier sheet
- **Edge cases:** Gmail sending limits.

#### Node: **Update follow up count in supplier sheet**
- **Type / role:** Google Sheets (Update) — tracks follow-up history.
- **Config choices:**
  - Matching column: `supplier_email`
  - Updates:
    - `last_follow_up = today`
    - `follow_up_count = parseInt(follow_up_count) + 1`
    - `follow_up_needed = "No"`
- **Expression detail:** Supplier email is taken from `$('Prepare message').item.json.supplier_email` (cross-node reference).
- **Outputs to:** No Operation, do nothing
- **Edge cases:**
  - If `follow_up_count` is blank/non-numeric, `parseInt('')` yields `NaN` in JS; here it uses `|| 0` earlier in Prepare message, but the update expression reads from `$json.follow_up_count` (current item), so ensure it is numeric-like.

#### Node: **No Operation, do nothing**
- **Type / role:** NoOp — terminator for follow-up branch.

---

### 2.5 Utility / Unused Nodes

#### Node: **No Operation, do nothing1** and **No Operation, do nothing**
- **Type / role:** NoOp — end markers; safe to remove without changing behavior.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Execute workflow | Manual Trigger | Start Module 1 manually | — | Hardcode suppliers details | Code in JavaScript hardcodes the details of the suppliers. This can however be customized by changing the node to fetch the details from different sources. |
| Hardcode suppliers details | Code | Emits supplier list items | Execute workflow | Reach out to suppliers | Code in JavaScript hardcodes the details of the suppliers. This can however be customized by changing the node to fetch the details from different sources. |
| Reach out to suppliers | Gmail | Send quote request emails | Hardcode suppliers details | Log to suppliers sheet | Reach out to suppliers and log details of suppliers to supplier_list sheet. |
| Log to suppliers sheet | Google Sheets | Append request tracking row | Reach out to suppliers | — | Reach out to suppliers and log details of suppliers to supplier_list sheet. |
| Trigger workflow every hour | Schedule Trigger | Start Module 2 periodically | — | Search for quote replies | Searches mail for responses from suppliers ever hour, if there is a response, node 2 extracts the response, parses the mail and looks out for an attachment. |
| Search for quote replies | Gmail | Find supplier reply emails | Trigger workflow every hour | Get email details | Searches mail for responses from suppliers ever hour, if there is a response, node 2 extracts the response, parses the mail and looks out for an attachment. |
| Get email details | Gmail | Fetch full email data by ID | Search for quote replies | Parse email data | Searches mail for responses from suppliers ever hour, if there is a response, node 2 extracts the response, parses the mail and looks out for an attachment. |
| Parse email data | Code | Normalize email fields + attachment heuristic | Get email details | Has attachments? | Searches mail for responses from suppliers ever hour, if there is a response, node 2 extracts the response, parses the mail and looks out for an attachment. |
| Has attachments? | IF | Gate update when attachments detected | Parse email data | Update supplier sheet (true) | If mail has attachment, the true branch returns an out put which is then used to update the quotes_received, follow-up needed and status columns in the supplier sheet. |
| Update supplier sheet | Google Sheets | Mark quote received in Supplier sheet | Has attachments? | — | If mail has attachment, the true branch returns an out put which is then used to update the quotes_received, follow-up needed and status columns in the supplier sheet. |
| Trigger workflow daily | Schedule Trigger | Start Module 3 (misnamed; configured hourly) | — | Get quotes to process | This reads the 'quote received' column of the supplier_list sheet... then download the attachment from that mail. |
| Get quotes to process | Google Sheets | Read suppliers where quote_received=Yes | Trigger workflow daily | Get supplier email | This reads the 'quote received' column of the supplier_list sheet... then download the attachment from that mail. |
| Get supplier email | Gmail | Search for emails with quote attachments | Get quotes to process | Download all attachments | This reads the 'quote received' column of the supplier_list sheet... then download the attachment from that mail. |
| Download all attachments | Gmail | Download attachments from message | Get supplier email | Extract from file | This reads the 'quote received' column of the supplier_list sheet... then download the attachment from that mail. |
| Extract from file | Extract From File | Convert PDF binary to text | Download all attachments | Prepare for openAI | Extract information from downloaded Pdf attachment\n\nObtain your API from https://platform.openai.com and connect your credentials |
| Prepare for openAI | Code | Aggregate text + attach supplier context | Extract from file | Information Extractor | Extract information from downloaded Pdf attachment\n\nObtain your API from https://platform.openai.com and connect your credentials |
| Extract key information from invoice | OpenAI Chat Model (LangChain) | LLM provider for extraction | — | Information Extractor (ai_languageModel) | Extract information from downloaded Pdf attachment\n\nObtain your API from https://platform.openai.com and connect your credentials |
| Information Extractor | Information Extractor (LangChain) | Extract structured fields from text | Prepare for openAI | Parse model output  | Extract information from downloaded Pdf attachment\n\nObtain your API from https://platform.openai.com and connect your credentials |
| Parse model output  | Code | Parse Products List JSON into rows | Information Extractor | Update supplier sheet with phone number; Save to Price Sheet | Parses extracted output to a valid json format, then updates the 'price comparison' sheet.\n\nAlso updates the supplier_list sheet with the phone number extracted from invoice for record keeping |
| Update supplier sheet with phone number | Google Sheets | Update supplier phone in Supplier sheet | Parse model output  | — | Parses extracted output to a valid json format, then updates the 'price comparison' sheet.\n\nAlso updates the supplier_list sheet with the phone number extracted from invoice for record keeping |
| Save to Price Sheet | Google Sheets | Append product pricing rows to Price sheet | Parse model output  | No Operation, do nothing1 | Parses extracted output to a valid json format, then updates the 'price comparison' sheet.\n\nAlso updates the supplier_list sheet with the phone number extracted from invoice for record keeping |
| No Operation, do nothing1 | NoOp | End marker | Save to Price Sheet | — |  |
| Trigger workflow daily1 | Schedule Trigger | Start Module 4 (misconfigured interval) | — | Get all quotes | Searches the supplier_list sheet to get a list of suppliers who needs a reminder to send their quote... |
| Get all quotes | Google Sheets | Read all suppliers | Trigger workflow daily1 | Filter pending quotes | Searches the supplier_list sheet to get a list of suppliers who needs a reminder to send their quote... |
| Filter pending quotes | Code | Select suppliers needing follow-up | Get all quotes | Any follow-ups needed? | Searches the supplier_list sheet to get a list of suppliers who needs a reminder to send their quote... |
| Any follow-ups needed? | IF | Check if follow-ups exist (currently flawed) | Filter pending quotes | Prepare message (true) | Searches the supplier_list sheet to get a list of suppliers who needs a reminder to send their quote... |
| Prepare message | Code | Build WhatsApp/email reminder + normalize phone | Any follow-ups needed? | Has phone number? | A customized message is prepared and sent to suppliers through to phone numbers or their email address if no phone number. |
| Has phone number? | IF | Route to WhatsApp vs email | Prepare message | Send WhatsApp follow-up message (true); Send follow-up mail (false) | A customized message is prepared and sent to suppliers through to phone numbers or their email address if no phone number. |
| Send WhatsApp follow-up message | Twilio | Send WhatsApp reminder | Has phone number? (true) | Update follow up count in supplier sheet | Configure Twilio WhatsApp Sandbox\n\nGo to Twilio Console → Messaging → Try it out → WhatsApp\nSend the join code from your phone (e.g., "join happy-elephant")\nCopy your sandbox number (e.g., +1 415 523 8886)\nUpdate "From" number in Send WhatsApp node |
| Send follow-up mail | Gmail | Send reminder via email | Has phone number? (false) | Update follow up count in supplier sheet | A customized message is prepared and sent to suppliers through to phone numbers or their email address if no phone number. |
| Update follow up count in supplier sheet | Google Sheets | Increment follow_up_count + set last_follow_up | Send WhatsApp follow-up message; Send follow-up mail | No Operation, do nothing | A customized message is prepared and sent to suppliers through to phone numbers or their email address if no phone number. |
| No Operation, do nothing | NoOp | End marker | Update follow up count in supplier sheet | — |  |
| Sticky Note4 | Sticky Note | Overview & setup instructions | — | — | ### Overview\nEliminate 90% of manual work in procurement by automating quote requests, response tracking, price extraction, and supplier follow-ups.\n\n### How it works\nThis workflow contains 4 independent automation modules that work together:\n Workflow 1: Quote Request Sender\nWorkflow 2: Response Monitor\nWorkflow 3: AI Price Extraction\nWorkflow 4: WhatsApp Follow-ups\n\n### How to set up\n#### 1. Create Google Sheets\nCreate "Supplier_list" sheet with columns: supplier_name \| supplier_email \| category \| request_date \| status \| quote_received \| phone_number \| last_follow_up \| follow_up_count\nCreate "Price Comparison" sheet with columns: supplier_name \| supplier_email \| product_name \| price \| currency \| quantity \| extracted_date \| source_file\nShare both sheets or enable API access\n\n#### 2. Connect API Credentials\nGmail OAuth: For sending/receiving emails (both Quote Sender and Response Monitor modules)\nGoogle Sheets OAuth: Use same account as Gmail (all modules)\nOpenAI API key: For AI price extraction (Module 3)\nTwilio Account: SID + Auth Token for WhatsApp (Module 4)\n\n#### 3. Configure Workflow\nUpdate all Google Sheet IDs in every Google Sheets node\nConfigure Gmail credentials on all Gmail nodes\nAdd OpenAI API key to HTTP Request node or Information Extractor\nConfigure Twilio credentials and join WhatsApp sandbox |
| Sticky Note1 / 2 / 3 / 5 / 6 / 7 / 8 / 9 / 10 / 11 / 12 / 13 | Sticky Note | Documentation only (no runtime) | — | — | (See individual notes in table rows above where applied) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Google Sheets**
   1. Create a spreadsheet for suppliers (e.g., “Supplier_sheet”) with a sheet tab containing headers:  
      `supplier_name, supplier_email, category, request_date, status, quote_received, follow_up_needed, phone_number, last_follow_up, follow_up_count`
   2. Create a spreadsheet for price comparison (e.g., “Price Comparison Sheet”) with headers:  
      `supplier_name, supplier_email, product_name, price, currency, quantity, notes, extracted_date, category, source_file`

2) **Create credentials in n8n**
   1. **Gmail OAuth2** credential (for sending/searching/downloading emails).
   2. **Google Sheets OAuth2** credential (access to both spreadsheets).
   3. **OpenAI API** credential (for LangChain OpenAI Chat Model).
   4. **Twilio API** credential (Account SID + Auth Token; WhatsApp-enabled number or sandbox).

3) **Module 1: Quote Request Sender**
   1. Add **Manual Trigger** node: `Execute workflow`.
   2. Add **Code** node: `Hardcode suppliers details`
      - Return an array of supplier objects `{name, email, category}` as separate items.
   3. Add **Gmail** node: `Reach out to suppliers`
      - Operation: **Send**
      - To: `{{$json.email}}`
      - Subject: `Price Quote Request - {{$json.category}}`
      - Body: HTML template; include a due date like `{{$now.plus(7,'days').toFormat('yyyy-MM-dd')}}`
   4. Add **Google Sheets** node: `Log to suppliers sheet`
      - Operation: **Append**
      - Spreadsheet: your Supplier sheet document ID
      - Map fields: supplier_name/email/category, request_date=today, follow_up_needed="Yes"
   5. Connect: Manual Trigger → Code → Gmail Send → Sheets Append

4) **Module 2: Response Monitoring**
   1. Add **Schedule Trigger** node: `Trigger workflow every hour` (hourly).
   2. Add **Gmail** node: `Search for quote replies`
      - Operation: **Get All**
      - Query: `subject:(Price Quote Request OR Re: Price Quote Request) newer_than:7d has:attachment`
      - Label: INBOX
   3. Add **Gmail** node: `Get email details`
      - Operation: **Get**
      - Message ID: `{{$json.id}}`
      - Simple: false
   4. Add **Code** node: `Parse email data`
      - Extract from/to/subject; produce `supplierEmail` and `hasAttachments`
   5. Add **IF** node: `Has attachments?`
      - Condition: boolean `{{$json.hasAttachments}}` is true
   6. Add **Google Sheets** node: `Update supplier sheet`
      - Operation: **Update**
      - Match on: `supplier_email`
      - Set: status="Quote Received", quote_received="Yes", follow_up_needed="No"
      - supplier_email from `{{$json.supplierEmail}}`
   7. Connect: Schedule → Gmail Search → Gmail Get → Code → IF → Sheets Update (true)

5) **Module 3: AI Price Extraction**
   1. Add **Schedule Trigger** node: `Trigger workflow daily`
      - Configure correctly for daily (in the provided JSON it’s hourly; change to days if intended).
   2. Add **Google Sheets** node: `Get quotes to process`
      - Operation: **Read/Get Many**
      - Filter: `quote_received == "Yes"`
   3. Add **Gmail** node: `Get supplier email`
      - Operation: **Get All**
      - Query: `subject:"Price Quote Request" has:attachment newer_than:7d`
      - Limit: 10
      - (Recommended improvement when rebuilding: include supplier email in query.)
   4. Add **Gmail** node: `Download all attachments`
      - Operation: **Get**
      - Message ID: `{{$json.id}}`
      - downloadAttachments: true
   5. Add **Extract From File** node: `Extract from file`
      - Operation: PDF
      - Binary property name: `{{ Object.keys($binary)[0] }}`
   6. Add **Code** node: `Prepare for openAI`
      - Concatenate extracted text into `document_text`
      - Add supplier metadata (supplier_name/email/category)
   7. Add **OpenAI Chat Model (LangChain)** node: `Extract key information from invoice`
      - Select your OpenAI credential
      - Choose a model (e.g., GPT-4o-mini) in the node UI if required by your n8n version
   8. Add **Information Extractor (LangChain)** node: `Information Extractor`
      - Text: `{{$json.document_text}}`
      - Attributes: include `Products List` (JSON string) and `supplier_phone`
      - Connect the OpenAI Chat Model to this node via the **AI Language Model** connection.
   9. Add **Code** node: `Parse model output`
      - Parse `output['Products List']` with `JSON.parse`
      - Output one item per product with pricing fields + supplier_phone
   10. Add **Google Sheets** node: `Save to Price Sheet`
       - Operation: Append
       - Spreadsheet: Price Comparison sheet document ID
       - Map product fields and supplier fields
   11. Add **Google Sheets** node: `Update supplier sheet with phone number`
       - Operation: Update (match `supplier_email`)
       - Set `phone_number` from extracted supplier_phone
   12. Connect: Schedule → Sheets Filter → Gmail Search → Gmail Get Attachments → Extract PDF → Prepare → Information Extractor → Parse → (to both Sheets nodes)

6) **Module 4: WhatsApp Follow-ups**
   1. Add **Schedule Trigger** node: `Trigger workflow daily1`
      - Configure a real cadence (daily). The provided JSON interval is not properly specified.
   2. Add **Google Sheets** node: `Get all quotes` (read all supplier rows).
   3. Add **Code** node: `Filter pending quotes`
      - Keep suppliers where quote_received != "Yes" and days since request is 3, 5, or 7.
   4. Add **IF** node: `Any follow-ups needed?`
      - Recommended when rebuilding: use `{{$items().length > 0}}` rather than `$json.length`.
   5. Add **Code** node: `Prepare message`
      - Build reminder message and normalize phone number.
   6. Add **IF** node: `Has phone number?`
      - Check `phone_number` is not empty.
   7. Add **Twilio** node: `Send WhatsApp follow-up message`
      - toWhatsapp = true
      - From = your Twilio WhatsApp number (sandbox or production)
      - To = `{{$json.phone_number}}`
      - Message = `{{$json.message}}`
   8. Add **Gmail** node: `Send follow-up mail` (fallback)
      - To = `{{$json.supplier_email}}`, body = `{{$json.message}}`
   9. Add **Google Sheets** node: `Update follow up count in supplier sheet`
      - Match on supplier_email
      - last_follow_up = today
      - follow_up_count = previous + 1
      - follow_up_needed = "No"
   10. Connect: Schedule → Sheets Read → Code Filter → IF → Code Prepare → IF → (Twilio or Gmail) → Sheets Update

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Obtain your API from https://platform.openai.com and connect your credentials” | OpenAI credential setup for AI extraction module |
| Twilio WhatsApp Sandbox steps: Console → Messaging → Try it out → WhatsApp; send join code; use sandbox number; update “From” in node | WhatsApp follow-ups configuration |
| Workflow overview + required Google Sheets columns + credential types + reminder to update all Sheet IDs in nodes | Sticky Note “Overview” block embedded in workflow |
| Disclaimer (provided by user): Le texte fourni provient exclusivement d’un workflow automatisé… | Compliance / provenance statement |

**Notable implementation risks to address when modifying:**
- Module 3 Gmail search is not constrained to the supplier being processed (can mix suppliers).
- Module 3 uses only the *first* supplier row context when multiple suppliers are pending.
- “Has attachments?” is keyword-based, even though Gmail search already required `has:attachment`.
- Module 4 “Any follow-ups needed?” IF condition is likely incorrect (`$json.length`).
- Two schedule triggers named “daily” are misconfigured in the JSON and should be fixed to true daily schedules if intended.