Send multi-stage overdue invoice reminders with NocoDB, Gmail, PostGrid and Slack

https://n8nworkflows.xyz/workflows/send-multi-stage-overdue-invoice-reminders-with-nocodb--gmail--postgrid-and-slack-14313


# Send multi-stage overdue invoice reminders with NocoDB, Gmail, PostGrid and Slack

# 1. Workflow Overview

This workflow automates multi-stage invoice follow-up for invoices stored in NocoDB. It runs daily, retrieves invoices and their related client records, identifies which invoices require action based on due date offsets, generates stage-specific HTML content, optionally sends a physical letter through PostGrid and/or an email through Gmail, and posts a tracking notification to Slack.

Typical use cases:
- Sending a friendly reminder before an invoice due date
- Sending a formal warning after the invoice becomes overdue
- Sending a legal-escalation notice after a longer overdue period
- Keeping an internal audit trail in Slack of what communication was sent

The workflow also includes a separate manual setup branch that creates the required NocoDB tables and the relation between them.

## 1.1 Scheduled Execution and Global Configuration

This block starts the daily automation and loads configurable business rules and company metadata used later in filtering and content generation.

## 1.2 Optional NocoDB Schema Bootstrap

This separate manual branch creates the required `Invoices` and `Clients` tables in NocoDB and links them together, so the main automation has the expected schema.

## 1.3 Invoice and Client Retrieval

This block fetches invoice rows from NocoDB, iterates through them, looks up the related client record, and combines both into one working item.

## 1.4 Invoice Eligibility Filtering and Stage Classification

This block removes invoices that should not be processed and splits the remaining invoices into three notification groups:
- upcoming/today reminder
- overdue warning
- legal notice

## 1.5 Message Rendering

This block creates the HTML body and email subject for each notification stage using invoice, client, config, and company data.

## 1.6 Delivery Routing and Internal Notification

This block conditionally sends a physical letter and/or email based on configuration flags, then posts a Slack message summarizing what was sent.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Execution and Global Configuration

### Overview
This block is the main entry path of the automation. It runs every day at a fixed hour, loads timing thresholds and delivery toggles, then loads the sender company details used in HTML templates and postal requests.

### Nodes Involved
- Run daily
- Config
- Your Company Details

### Node Details

#### Run daily
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow trigger that starts the main process on a schedule.
- **Configuration choices:** Configured to run daily at hour `7`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point of the main branch; outputs to `Config`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Time zone differences between server and business expectation
  - Missed executions if n8n instance is down at scheduled time

#### Config
- **Type and technical role:** `n8n-nodes-base.set`; stores runtime configuration values.
- **Configuration choices:** Defines:
  - `Days before Due Date for reminder` = `7`
  - `Days after Due Date for warning` = `7`
  - `Days after Due Date for Legal action` = `10`
  - `Send Email` = `true`
  - `Send Physical Letter` = `true`
- **Key expressions or variables used:** Referenced later via expressions like:
  - `$('Config').first().json['Days before Due Date for reminder']`
  - `$('Config').item.json["Send Email"]`
- **Input and output connections:** Input from `Run daily`; output to `Your Company Details`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Incorrect values can suppress or prematurely trigger notices
  - Booleans set incorrectly may stop all communications
  - Negative or non-business-valid thresholds may produce unexpected filtering

#### Your Company Details
- **Type and technical role:** `n8n-nodes-base.set`; stores sender/company metadata.
- **Configuration choices:** Defines placeholders for:
  - Company Name
  - Email
  - Phone
  - Country Code
  - City
  - Street
  - Postal Code
  - Swift Code
  - Bank Name
  - Bank Account Number
  - TIN
- **Key expressions or variables used:** Reused widely in HTML templates and PostGrid request body, e.g.:
  - `$('Your Company Details').first().json['Company Name']`
  - `$('Your Company Details').first().json['Bank Name']`
- **Input and output connections:** Input from `Config`; output to `Get Invoices`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Empty company fields produce incomplete emails/letters
  - Invalid country/postal/address values may cause postal API rejection
  - Missing banking details weaken payment instructions

---

## 2.2 Optional NocoDB Schema Bootstrap

### Overview
This is a separate manual branch used once during setup. It creates the `Invoices` and `Clients` tables in NocoDB and then creates a relation field between them.

### Nodes Involved
- Create Tables
- NocoDB Config
- Create NocoDB Invoices Table
- Create NocoDB Clients Table
- Create relation between tables

### Node Details

#### Create Tables
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for schema setup.
- **Configuration choices:** No extra parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point of setup branch; outputs to `NocoDB Config`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Must be run manually
  - Should not be re-run blindly if tables already exist

#### NocoDB Config
- **Type and technical role:** `n8n-nodes-base.set`; stores NocoDB base metadata for setup API calls.
- **Configuration choices:** Defines:
  - `Base ID`
  - `NocoDB Url`
- **Key expressions or variables used:** Later used in URLs like:
  - `$('NocoDB Config').first().json['NocoDB Url']`
  - `$('NocoDB Config').first().json['Base ID']`
- **Input and output connections:** Input from `Create Tables`; output to `Create NocoDB Invoices Table`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Empty or malformed base URL causes HTTP failures
  - Wrong Base ID creates resources in the wrong base or returns 404

#### Create NocoDB Invoices Table
- **Type and technical role:** `n8n-nodes-base.httpRequest`; direct API call to NocoDB metadata API to create the `Invoices` table.
- **Configuration choices:**
  - `POST` to `/api/v3/meta/bases/{Base ID}/tables`
  - Auth via predefined `nocoDbApiToken`
  - Sends JSON body describing fields:
    - Invoice Id
    - Amount (Currency, code PLN)
    - Issue Date
    - Due Date
    - Invoice Status (Unpaid, Partial, Paid, Overdue)
    - Vindication Status (None, Reminder Sent, Warning Sent, Legal Notice Sent)
    - Reminder/Warning/Legal Notice Sent Date
    - Payment Date
- **Key expressions or variables used:**
  - URL uses `NocoDB Url` and `Base ID`
- **Input and output connections:** Input from `NocoDB Config`; output to `Create NocoDB Clients Table`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Existing table name conflict
  - Invalid NocoDB auth token
  - NocoDB API version mismatch
  - Currency code hardcoded to `PLN`, which may not fit all installations

#### Create NocoDB Clients Table
- **Type and technical role:** `n8n-nodes-base.httpRequest`; creates the `Clients` table in NocoDB.
- **Configuration choices:**
  - `POST` to same metadata endpoint
  - Auth via predefined `nocoDbApiToken`
  - Creates fields:
    - Name
    - Email
    - Country Code
    - Province
    - City
    - Postal Code
    - Address
  - Includes `"source_id": "{{ $json.source_id }}"` from previous response
- **Key expressions or variables used:**
  - URL uses values from `NocoDB Config`
  - Body references previous node response field `source_id`
- **Input and output connections:** Input from `Create NocoDB Invoices Table`; output to `Create relation between tables`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Existing table name conflict
  - `source_id` dependency on prior API response shape
  - Invalid auth or wrong base metadata permissions

#### Create relation between tables
- **Type and technical role:** `n8n-nodes-base.httpRequest`; adds a relation field to link clients to invoices.
- **Configuration choices:**
  - `POST` to `/tables/{clientTableId}/fields`
  - Creates field titled `Invoices`
  - `type = Links`
  - `relation_type = hm`
  - `related_table_id` references the created `Invoices` table ID
- **Key expressions or variables used:**
  - `{{ $json.id }}` from `Create NocoDB Clients Table`
  - `$('Create NocoDB Invoices Table').first().json['id']`
- **Input and output connections:** Input from `Create NocoDB Clients Table`; no downstream node.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Wrong relation semantics if schema expectations differ
  - Fails if either table creation failed upstream
  - Auth/permission problems on metadata endpoint

---

## 2.3 Invoice and Client Retrieval

### Overview
This block fetches invoice records from NocoDB, iterates through them one by one, looks up each related client using the invoice’s foreign key, and produces a combined item with both invoice and client objects.

### Nodes Involved
- Get Invoices
- Get client info for invoice
- Get client row
- Add client info to invoice

### Node Details

#### Get Invoices
- **Type and technical role:** `n8n-nodes-base.nocoDb`; retrieves all rows from the invoices table.
- **Configuration choices:**
  - Operation: `getAll`
  - Project/Base ID: `pksfpoc943gwhvy`
  - Table ID: `mi7ptepxqhw71hw`
  - `returnAll = true`
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:** Input from `Your Company Details`; output to `Get client info for invoice`.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Hardcoded project and table IDs mean this node must be updated in a new environment
  - Empty table results in no downstream processing
  - NocoDB auth failures or network errors

#### Get client info for invoice
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates invoices so the workflow can fetch each related client.
- **Configuration choices:** Default batch behavior with no custom options.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Receives all invoices from `Get Invoices`
  - Main output 0 sends each item to `Get only unpaid invoices`
  - Loop output 1 sends current invoice to `Get client row`
  - `Add client info to invoice` loops back into this node
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Incorrect loop wiring can skip or repeat items
  - Large datasets may take long to process
  - If a downstream node errors for one item, batch progression may halt

#### Get client row
- **Type and technical role:** `n8n-nodes-base.nocoDb`; retrieves the related client row by ID.
- **Configuration choices:**
  - Fetches a row from client table `mjsltidf4t9ex0m`
  - Uses row ID expression `={{ $json.Clients_id }}`
  - Project/Base ID `pksfpoc943gwhvy`
- **Key expressions or variables used:**
  - `$json.Clients_id`
- **Input and output connections:** Input from `Get client info for invoice` loop path; output to `Add client info to invoice`.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Assumes each invoice contains `Clients_id`
  - Missing or invalid client ID causes fetch failure
  - Hardcoded table ID must match target NocoDB schema

#### Add client info to invoice
- **Type and technical role:** `n8n-nodes-base.set`; builds a combined payload for downstream logic.
- **Configuration choices:**
  - Sets `Invoice` to the current invoice item from `Get client info for invoice`
  - Sets `Client` to the fetched client row from current input
- **Key expressions or variables used:**
  - `={{ $('Get client info for invoice').item.json }}`
  - `={{ $json }}`
- **Input and output connections:** Input from `Get client row`; output back to `Get client info for invoice` to continue loop.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Expression scope confusion if node names change
  - If client fetch failed, object structure is incomplete
  - Output shape differs from raw NocoDB records, so downstream expressions depend on this exact structure
- **Sub-workflow reference:** None.

---

## 2.4 Invoice Eligibility Filtering and Stage Classification

### Overview
This block decides which combined invoice/client items need action today. It first applies a broad unpaid/eligible filter, then fans out the item into one or more stage-specific filters based on due date difference.

### Nodes Involved
- Get only unpaid invoices
- Get todays / upcoming invoices
- Get overdue invoices for warning
- Get overdue invoices for Legal notice

### Node Details

#### Get only unpaid invoices
- **Type and technical role:** `n8n-nodes-base.filter`; screens out invoices that should not be processed.
- **Configuration choices:** Uses `or` combinator across three conditions:
  - Invoice Status is not `Paid`
  - Due Date is before `$now - reminder_days`
  - Payment Date does not exist
- **Key expressions or variables used:**
  - `={{ $json.Invoice["Invoice Status"] }}`
  - `={{ Date($json.Invoice["Due Date"]) }}`
  - `={{ $now.minus($('Config').first().json['Days before Due Date for reminder'],'days') }}`
  - `={{ $json.Invoice["Payment Date"] }}`
- **Input and output connections:** Input from `Get client info for invoice` output 0; outputs in parallel to all three stage filters.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Logical design uses `or`, which is permissive; many items may pass if any one condition matches
  - If intent was “not paid AND no payment date AND relevant date”, this node currently does not enforce that strictly
  - Date parsing via `Date(...)` may behave differently by locale/runtime
  - Missing `Invoice` object breaks expressions

#### Get todays / upcoming invoices
- **Type and technical role:** `n8n-nodes-base.filter`; identifies reminder-stage invoices.
- **Configuration choices:** Uses `or`:
  - Due date difference equals configured reminder offset
  - Due date equals today
- **Key expressions or variables used:**
  - `={{ $json.Invoice["Due Date"].toDateTime().diffTo($now.format('yyyy-LL-dd'), 'days') }}`
  - `={{ $('Config').first().json['Days before Due Date for reminder'] }}`
  - `={{ $now.format('yyyy-LL-dd') }}`
- **Input and output connections:** Input from `Get only unpaid invoices`; output to `Prepare Reminder Mail and Letter HTML`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - `diffTo` sign conventions must match expected due-date math
  - Comparing date string to formatted date assumes matching format
  - Time zone or midnight boundary issues can cause off-by-one results

#### Get overdue invoices for warning
- **Type and technical role:** `n8n-nodes-base.filter`; identifies invoices overdue by exactly the warning threshold.
- **Configuration choices:** Passes items where day difference equals negative configured warning days.
- **Key expressions or variables used:**
  - `={{ $json.Invoice["Due Date"].toDateTime().diffTo($now.format('yyyy-LL-dd'), 'days') }}`
  - `={{ -$('Config').first().json['Days after Due Date for warning'] }}`
- **Input and output connections:** Input from `Get only unpaid invoices`; output to `Prepare Pre-trial summon Mail and Letter HTML`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Only exact match is processed; if daily run is missed, the warning may never send
  - Invalid/missing due date breaks date calculation

#### Get overdue invoices for Legal notice
- **Type and technical role:** `n8n-nodes-base.filter`; identifies invoices overdue by exactly the legal-action threshold.
- **Configuration choices:** Passes items where day difference equals negative legal threshold.
- **Key expressions or variables used:**
  - `={{ $json.Invoice["Due Date"].toDateTime().diffTo($now.format('yyyy-LL-dd'), 'days') }}`
  - `={{ -$('Config').first().json['Days after Due Date for Legal action'] }}`
- **Input and output connections:** Input from `Get only unpaid invoices`; output to `Prepare Court summon Mail and Letter HTML`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Same “exact day only” risk as warning branch
  - If the schedule does not run on that exact date, the legal notice is skipped

---

## 2.5 Message Rendering

### Overview
This block transforms each eligible item into a notification payload containing stage-specific HTML and an email subject. All branches then pass through a simple normalization node before delivery routing.

### Nodes Involved
- Prepare Reminder Mail and Letter HTML
- Prepare Pre-trial summon Mail and Letter HTML
- Prepare Court summon Mail and Letter HTML
- Combine all inputs

### Node Details

#### Prepare Reminder Mail and Letter HTML
- **Type and technical role:** `n8n-nodes-base.set`; generates a friendly reminder HTML document and subject.
- **Configuration choices:**
  - Adds `Message Content` containing full HTML
  - Adds `Email Subject` = `Reminder to pay invoice {{ Invoice Id }}`
  - `includeOtherFields = true`
- **Key expressions or variables used:**
  - Client name via `$json['Name (from Invoice Clients)']`
  - Invoice fields via `$json.Invoice[...]`
  - Company fields via `$('Your Company Details').first().json[...]`
  - Config threshold via `$('Config').first().json[...]`
- **Input and output connections:** Input from `Get todays / upcoming invoices`; output to `Combine all inputs`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Uses `$json['Name (from Invoice Clients)']`, while combined payload mainly contains `Client`; this may rely on retained original fields from NocoDB and can fail if field names differ
  - Large HTML content may be harder to maintain
  - Missing company or client fields degrade message quality

#### Prepare Pre-trial summon Mail and Letter HTML
- **Type and technical role:** `n8n-nodes-base.set`; generates a stronger warning/pre-legal notice.
- **Configuration choices:**
  - Adds `Message Content` containing warning HTML
  - Adds `Email Subject` = `Will to enter legal route over invoice {{ Invoice Id }}`
  - `includeOtherFields = true`
- **Key expressions or variables used:** Same expression patterns as above, including invoice and company fields.
- **Input and output connections:** Input from `Get overdue invoices for warning`; output to `Combine all inputs`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Same potential field-name mismatch for recipient name
  - Legal wording may need jurisdiction review before production use

#### Prepare Court summon Mail and Letter HTML
- **Type and technical role:** `n8n-nodes-base.set`; generates the final legal-escalation notice.
- **Configuration choices:**
  - Adds `Message Content` with litigation-themed HTML
  - Adds `Email Subject` = `Entering Legal route over invoice {{ Invoice Id }}`
  - `includeOtherFields = true`
- **Key expressions or variables used:** Invoice values, company values, and recipient name expressions.
- **Input and output connections:** Input from `Get overdue invoices for Legal notice`; output to `Combine all inputs`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Legal notice content may be inappropriate or risky without legal approval
  - Same naming/dependency issues as other templates

#### Combine all inputs
- **Type and technical role:** `n8n-nodes-base.set`; pass-through normalization node.
- **Configuration choices:** `includeOtherFields = true`; no additional assignments.
- **Key expressions or variables used:** None.
- **Input and output connections:** Inputs from all three template nodes; outputs in parallel to `Send Letter?` and `Send Email?`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Mostly cosmetic; if removed, delivery logic would still likely work with direct connections
  - Parallel arrivals from multiple branches are treated as separate items, not merged records

---

## 2.6 Delivery Routing and Internal Notification

### Overview
This block decides whether to send a physical letter and/or email according to config flags. After either delivery channel completes, Slack receives an internal summary message.

### Nodes Involved
- Send Letter?
- Send Letter using PostGrid API
- Send Email?
- Send an Email
- Send a message

### Node Details

#### Send Letter?
- **Type and technical role:** `n8n-nodes-base.if`; enables or disables postal sending.
- **Configuration choices:** True branch when `Config["Send Physical Letter"]` is `true`.
- **Key expressions or variables used:**
  - `={{ $('Config').item.json["Send Physical Letter"] }}`
- **Input and output connections:** Input from `Combine all inputs`; true output to `Send Letter using PostGrid API`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Uses `.item` rather than `.first()`, which depends on item linkage behavior
  - If config reference cannot resolve, all letters may be skipped

#### Send Letter using PostGrid API
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a postal letter request to PostGrid.
- **Configuration choices:**
  - `POST https://api.postgrid.com/print-mail/v1/letters`
  - Uses generic custom auth with API key header
  - Builds `to` address from `Client`
  - Builds `from` address from `Your Company Details`
  - Sends HTML content from `Message Content`
  - Includes description with invoice ID
  - `color = false`, `doubleSided = false`, `addressPlacement = top_first_page`
- **Key expressions or variables used:**
  - `{{ $json.Client.Name }}`
  - `{{ $json.Client.Address }}`
  - `{{ $json.Client["Postal Code"] }}`
  - `{{JSON.stringify($json['Message Content'])}}`
  - Company fields from `Your Company Details`
- **Input and output connections:** Input from `Send Letter?`; output to `Send a message`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Invalid postal address fields may be rejected by PostGrid
  - Test API key may not create production mail
  - HTML size/format limitations
  - Missing client country code, province, or postal code can break letter creation
- **Sub-workflow reference:** None.

#### Send Email?
- **Type and technical role:** `n8n-nodes-base.if`; enables or disables email sending.
- **Configuration choices:** True branch when `Config["Send Email"]` is `true`.
- **Key expressions or variables used:**
  - `={{ $('Config').item.json["Send Email"] }}`
- **Input and output connections:** Input from `Combine all inputs`; true output to `Send an Email`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Same `.item` reference concern as `Send Letter?`
  - If false, Slack notice will only occur from the letter branch

#### Send an Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the HTML email via Gmail.
- **Configuration choices:**
  - To: `{{ $json.Client.Email }}`
  - Subject: `{{ $json["Email Subject"] }}`
  - Message/body: `{{ $json["Message Content"] }}`
  - Auth via Gmail OAuth2
- **Key expressions or variables used:**
  - Client email
  - Generated subject and HTML body
- **Input and output connections:** Input from `Send Email?`; output to `Send a message`.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Gmail OAuth expiry/revocation
  - Invalid recipient email
  - Gmail sending quotas or anti-spam limits
  - HTML rendering may vary by client

#### Send a message
- **Type and technical role:** `n8n-nodes-base.slack`; posts an internal Slack block message about the sent communication.
- **Configuration choices:**
  - Sends to a selected channel
  - Message type: block
  - Uses structured block JSON summarizing:
    - Email subject/message type
    - Invoice number
    - Days overdue
    - Contractor/client name
    - Payment deadline
    - Amount
    - Issue date
- **Key expressions or variables used:**
  - `$('Combine all inputs').item.json['Email Subject']`
  - `$( "Combine all inputs" ).item.json.Invoice["Invoice Id"]`
  - `$now.diffTo($( "Combine all inputs" ).item.json.Invoice["Due Date"].toDateTime(), 'days').floor()`
  - `$( "Combine all inputs" ).item.json.Client["Name"]`
- **Input and output connections:** Input from either `Send an Email` or `Send Letter using PostGrid API`; no downstream node.
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases or potential failure types:**
  - If both email and letter are enabled, Slack may receive two messages for the same invoice/stage
  - Slack OAuth/channel access issues
  - Block JSON expression syntax is sensitive to quoting
  - “Overdue” calculation may display negative or counterintuitive values for upcoming reminders

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run daily | Schedule Trigger | Daily entry point for invoice processing |  | Config | ## Run daily |
| Config | Set | Stores timing thresholds and delivery toggles | Run daily | Your Company Details | ## Fill out configs<br>"Config" node describes when notifications should be sent as well if autmation should send Email, letter, or both.<br><br>"Your Company Details" node is necessary for contents of both Email and letter. Make sure that provided data is correct |
| Your Company Details | Set | Stores sender company identity and payment details | Config | Get Invoices | ## Fill out configs<br>"Config" node describes when notifications should be sent as well if autmation should send Email, letter, or both.<br><br>"Your Company Details" node is necessary for contents of both Email and letter. Make sure that provided data is correct |
| Create Tables | Manual Trigger | Manual setup entry point for NocoDB schema creation |  | NocoDB Config | ## Set up NocoDB Tables first!<br>This short 'script' will create necessary tables inside NocoDB for you. One thing less to worry about. Make sure you don't have table called `Clients` or `Invoices` |
| NocoDB Config | Set | Stores NocoDB base URL and base ID for schema bootstrap | Create Tables | Create NocoDB Invoices Table | ## Set up NocoDB Tables first!<br>This short 'script' will create necessary tables inside NocoDB for you. One thing less to worry about. Make sure you don't have table called `Clients` or `Invoices` |
| Create NocoDB Invoices Table | HTTP Request | Creates the Invoices table in NocoDB | NocoDB Config | Create NocoDB Clients Table | ## Set up NocoDB Tables first!<br>This short 'script' will create necessary tables inside NocoDB for you. One thing less to worry about. Make sure you don't have table called `Clients` or `Invoices` |
| Create NocoDB Clients Table | HTTP Request | Creates the Clients table in NocoDB | Create NocoDB Invoices Table | Create relation between tables | ## Set up NocoDB Tables first!<br>This short 'script' will create necessary tables inside NocoDB for you. One thing less to worry about. Make sure you don't have table called `Clients` or `Invoices` |
| Create relation between tables | HTTP Request | Adds relation field linking clients to invoices | Create NocoDB Clients Table |  | ## Set up NocoDB Tables first!<br>This short 'script' will create necessary tables inside NocoDB for you. One thing less to worry about. Make sure you don't have table called `Clients` or `Invoices` |
| Get Invoices | NocoDB | Fetches all invoice rows | Your Company Details | Get client info for invoice | ## Step 1: Fetch invoices and client info from NocoDB<br>In this step we fetch invoice data from NocoDB, then we find matching client for each invoice and merge both together |
| Get client info for invoice | Split In Batches | Iterates invoices for client lookup and loop control | Get Invoices, Add client info to invoice | Get only unpaid invoices, Get client row | ## Step 1: Fetch invoices and client info from NocoDB<br>In this step we fetch invoice data from NocoDB, then we find matching client for each invoice and merge both together |
| Get client row | NocoDB | Fetches client row linked to current invoice | Get client info for invoice | Add client info to invoice | ## Step 1: Fetch invoices and client info from NocoDB<br>In this step we fetch invoice data from NocoDB, then we find matching client for each invoice and merge both together |
| Add client info to invoice | Set | Combines invoice and client into one payload | Get client row | Get client info for invoice | ## Step 1: Fetch invoices and client info from NocoDB<br>In this step we fetch invoice data from NocoDB, then we find matching client for each invoice and merge both together |
| Get only unpaid invoices | Filter | Broad eligibility filter for invoices to process | Get client info for invoice | Get todays / upcoming invoices, Get overdue invoices for warning, Get overdue invoices for Legal notice | ## Step 2: Filter out invoices we don't care about<br>We filter out invoices based on couple values like:<br>- Invoice Status - Must not be "Paid"<br>- Payment Date - Must not be specified<br>- Due Date - Combined with config we check if date in the invoice is today or later<br><br>Then we split remaining invoices into 3 groups:<br>- Today/Incoming - These will result in reminder to pay<br>- For warning - These will result in a message about will to enter legal route<br>- For legal notice - These will result in a message about fact that we entered legal route |
| Get todays / upcoming invoices | Filter | Classifies reminder-stage invoices | Get only unpaid invoices | Prepare Reminder Mail and Letter HTML | ## Step 2: Filter out invoices we don't care about<br>We filter out invoices based on couple values like:<br>- Invoice Status - Must not be "Paid"<br>- Payment Date - Must not be specified<br>- Due Date - Combined with config we check if date in the invoice is today or later<br><br>Then we split remaining invoices into 3 groups:<br>- Today/Incoming - These will result in reminder to pay<br>- For warning - These will result in a message about will to enter legal route<br>- For legal notice - These will result in a message about fact that we entered legal route |
| Get overdue invoices for warning | Filter | Classifies warning-stage overdue invoices | Get only unpaid invoices | Prepare Pre-trial summon Mail and Letter HTML | ## Step 2: Filter out invoices we don't care about<br>We filter out invoices based on couple values like:<br>- Invoice Status - Must not be "Paid"<br>- Payment Date - Must not be specified<br>- Due Date - Combined with config we check if date in the invoice is today or later<br><br>Then we split remaining invoices into 3 groups:<br>- Today/Incoming - These will result in reminder to pay<br>- For warning - These will result in a message about will to enter legal route<br>- For legal notice - These will result in a message about fact that we entered legal route |
| Get overdue invoices for Legal notice | Filter | Classifies final legal-stage overdue invoices | Get only unpaid invoices | Prepare Court summon Mail and Letter HTML | ## Step 2: Filter out invoices we don't care about<br>We filter out invoices based on couple values like:<br>- Invoice Status - Must not be "Paid"<br>- Payment Date - Must not be specified<br>- Due Date - Combined with config we check if date in the invoice is today or later<br><br>Then we split remaining invoices into 3 groups:<br>- Today/Incoming - These will result in reminder to pay<br>- For warning - These will result in a message about will to enter legal route<br>- For legal notice - These will result in a message about fact that we entered legal route |
| Prepare Reminder Mail and Letter HTML | Set | Builds reminder-stage HTML and subject | Get todays / upcoming invoices | Combine all inputs | ## Step 3: Prepare Email / Letter contents<br>In this step you can paste your own HTML and use data from previous nodes to customize it.<br><br>Email Subject is used not only in Email, but also in Slack Notification. This will make it easier to know what given notification really is about<br><br>At the end we combine all 3 inputs into one. It server only aesthetic purpose (increases readability) |
| Prepare Pre-trial summon Mail and Letter HTML | Set | Builds warning-stage HTML and subject | Get overdue invoices for warning | Combine all inputs | ## Step 3: Prepare Email / Letter contents<br>In this step you can paste your own HTML and use data from previous nodes to customize it.<br><br>Email Subject is used not only in Email, but also in Slack Notification. This will make it easier to know what given notification really is about<br><br>At the end we combine all 3 inputs into one. It server only aesthetic purpose (increases readability) |
| Prepare Court summon Mail and Letter HTML | Set | Builds legal-stage HTML and subject | Get overdue invoices for Legal notice | Combine all inputs | ## Step 3: Prepare Email / Letter contents<br>In this step you can paste your own HTML and use data from previous nodes to customize it.<br><br>Email Subject is used not only in Email, but also in Slack Notification. This will make it easier to know what given notification really is about<br><br>At the end we combine all 3 inputs into one. It server only aesthetic purpose (increases readability) |
| Combine all inputs | Set | Pass-through node to normalize stage outputs | Prepare Reminder Mail and Letter HTML, Prepare Pre-trial summon Mail and Letter HTML, Prepare Court summon Mail and Letter HTML | Send Letter?, Send Email? | ## Step 3: Prepare Email / Letter contents<br>In this step you can paste your own HTML and use data from previous nodes to customize it.<br><br>Email Subject is used not only in Email, but also in Slack Notification. This will make it easier to know what given notification really is about<br><br>At the end we combine all 3 inputs into one. It server only aesthetic purpose (increases readability) |
| Send Letter? | If | Checks whether physical letters are enabled | Combine all inputs | Send Letter using PostGrid API | ## Step 4: Send a notification<br>In this step we send notification to the client, and notification to Slack channel about it so that we can keep track of what invoices has been processed |
| Send Letter using PostGrid API | HTTP Request | Sends postal letter via PostGrid | Send Letter? | Send a message | ## Step 4: Send a notification<br>In this step we send notification to the client, and notification to Slack channel about it so that we can keep track of what invoices has been processed |
| Send Email? | If | Checks whether email sending is enabled | Combine all inputs | Send an Email | ## Step 4: Send a notification<br>In this step we send notification to the client, and notification to Slack channel about it so that we can keep track of what invoices has been processed |
| Send an Email | Gmail | Sends HTML email to the client | Send Email? | Send a message | ## Step 4: Send a notification<br>In this step we send notification to the client, and notification to Slack channel about it so that we can keep track of what invoices has been processed |
| Send a message | Slack | Posts internal Slack notification about sent communication | Send an Email, Send Letter using PostGrid API |  | ## Step 4: Send a notification<br>In this step we send notification to the client, and notification to Slack channel about it so that we can keep track of what invoices has been processed |
| Sticky Note | Sticky Note | Documentation note on setup and integrations |  |  | ## Automatic vindication of overdue or incoming invoices<br><br>### How it works<br><br>This workflow automates sending reminders about invoices that due date is comming soon.<br>Its using [PostGrid](https://www.postgrid.com) for traditional mail, [NocoDB](https://nocodb.com) for storing information about invoices and Gmail for sending E-mails, as well as Slack for notifying user (you!) about notifications that has been sent. <br><br>### Set up<br><br>- Fill out "Config" node containing all time spans when each type of notification should be sent<br>- Fill out "Your Company Details" node containing information about your company. It will be used in E-mail/letter templates as well as used in API call to PostGrid.<br>- Prepare auth for "Send Letter using PostGrid API" node.<br>  - Use "Custom Auth" and paste:<br>```<br>{<br>	"headers": {<br>		"x-api-key": "<Your API Key>"<br>	}<br>}<br>```<br><br>### Customize<br><br>If you already have tables "Clients" or "Invoices" in NocoDB, you can use different names. In order to do that, you must edit:<br>1. JSON Body in "Create NocoDB Invoices Table" and "Create NocoDB Clients Table" nodes<br>2. Row ID Value in "Get client row" node<br><br><br>Need help? Contact us at developers@sailingbyte.com<br><br>Happy hacking! |
| Sticky Note1 | Sticky Note | Visual label for daily schedule area |  |  | ## Run daily |
| Sticky Note2 | Sticky Note | Visual note about required config fields |  |  | ## Fill out configs<br>"Config" node describes when notifications should be sent as well if autmation should send Email, letter, or both.<br><br>"Your Company Details" node is necessary for contents of both Email and letter. Make sure that provided data is correct |
| Sticky Note4 | Sticky Note | Visual note about initial NocoDB setup |  |  | ## Set up NocoDB Tables first!<br>This short 'script' will create necessary tables inside NocoDB for you. One thing less to worry about. Make sure you don't have table called `Clients` or `Invoices` |
| Sticky Note5 | Sticky Note | Visual note for invoice/client retrieval phase |  |  | ## Step 1: Fetch invoices and client info from NocoDB<br>In this step we fetch invoice data from NocoDB, then we find matching client for each invoice and merge both together |
| Sticky Note6 | Sticky Note | Visual note for filtering/classification phase |  |  | ## Step 2: Filter out invoices we don't care about<br>We filter out invoices based on couple values like:<br>- Invoice Status - Must not be "Paid"<br>- Payment Date - Must not be specified<br>- Due Date - Combined with config we check if date in the invoice is today or later<br><br>Then we split remaining invoices into 3 groups:<br>- Today/Incoming - These will result in reminder to pay<br>- For warning - These will result in a message about will to enter legal route<br>- For legal notice - These will result in a message about fact that we entered legal route |
| Sticky Note7 | Sticky Note | Visual note for content generation phase |  |  | ## Step 3: Prepare Email / Letter contents<br>In this step you can paste your own HTML and use data from previous nodes to customize it.<br><br>Email Subject is used not only in Email, but also in Slack Notification. This will make it easier to know what given notification really is about<br><br>At the end we combine all 3 inputs into one. It server only aesthetic purpose (increases readability) |
| Sticky Note8 | Sticky Note | Visual note for delivery phase |  |  | ## Step 4: Send a notification<br>In this step we send notification to the client, and notification to Slack channel about it so that we can keep track of what invoices has been processed |

---

# 4. Reproducing the Workflow from Scratch

## A. Create the setup branch for NocoDB schema creation

1. **Create a `Manual Trigger` node** named `Create Tables`.

2. **Create a `Set` node** named `NocoDB Config`.
   - Add string fields:
     - `Base ID`
     - `NocoDB Url`
   - Fill them with your NocoDB base identifier and root URL, for example:
     - `https://your-nocodb-instance/`

3. **Connect `Create Tables` → `NocoDB Config`.**

4. **Create an `HTTP Request` node** named `Create NocoDB Invoices Table`.
   - Method: `POST`
   - URL:
     - `={{ $json["NocoDB Url"] }}api/v3/meta/bases/{{$('NocoDB Config').first().json['Base ID']}}/tables`
   - Authentication: `Predefined Credential Type`
   - Credential type: `NocoDB API Token`
   - Body type: JSON
   - Enable `Send Body`
   - Add a JSON body defining the `Invoices` table with these fields:
     - Invoice Id: SingleLineText
     - Amount: Currency (`PLN` in this template)
     - Issue Date: Date
     - Due Date: Date
     - Invoice Status: SingleSelect with `Unpaid`, `Partial`, `Paid`, `Overdue`
     - Vindication Status: SingleSelect with `None`, `Reminder Sent`, `Warning Sent`, `Legal Notice Sent`
     - Reminder Sent Date: Date
     - Warning Sent Date: Date
     - Legal Notice Sent Date: Date
     - Payment Date: Date

5. **Connect `NocoDB Config` → `Create NocoDB Invoices Table`.**

6. **Create another `HTTP Request` node** named `Create NocoDB Clients Table`.
   - Method: `POST`
   - URL:
     - `={{ $('NocoDB Config').first().json['NocoDB Url'] }}api/v3/meta/bases/{{$('NocoDB Config').first().json['Base ID']}}/tables`
   - Authentication: same NocoDB API token
   - Body type: JSON
   - Enable `Send Body`
   - JSON body should create a `Clients` table with fields:
     - Name
     - Email
     - Country Code
     - Province
     - City
     - Postal Code
     - Address
   - Include `source_id` from the prior response if your NocoDB metadata API expects it:
     - `{{ $json.source_id }}`

7. **Connect `Create NocoDB Invoices Table` → `Create NocoDB Clients Table`.**

8. **Create a third `HTTP Request` node** named `Create relation between tables`.
   - Method: `POST`
   - URL:
     - `={{ $('NocoDB Config').first().json['NocoDB Url'] }}api/v3/meta/bases/{{$('NocoDB Config').first().json['Base ID']}}/tables/{{ $json.id }}/fields`
   - Authentication: same NocoDB API token
   - Body type: JSON
   - JSON body:
     - title: `Invoices`
     - type: `Links`
     - options:
       - relation_type: `hm`
       - related_table_id: ID from `Create NocoDB Invoices Table`

9. **Connect `Create NocoDB Clients Table` → `Create relation between tables`.**

10. **Run the setup branch manually once**.
   - Do this only if the tables do not already exist.
   - If you already have tables, skip this branch and instead update the hardcoded table IDs in the main branch.

---

## B. Create the main scheduled automation branch

11. **Create a `Schedule Trigger` node** named `Run daily`.
   - Configure it to run every day at hour `7`.
   - Adjust timezone settings in n8n if needed.

12. **Create a `Set` node** named `Config`.
   - Add:
     - `Days before Due Date for reminder` as Number = `7`
     - `Days after Due Date for warning` as Number = `7`
     - `Days after Due Date for Legal action` as Number = `10`
     - `Send Email` as Boolean = `true`
     - `Send Physical Letter` as Boolean = `true`

13. **Connect `Run daily` → `Config`.**

14. **Create a `Set` node** named `Your Company Details`.
   - Add string fields:
     - Company Name
     - Email
     - Phone
     - Country Code
     - City
     - Street
     - Postal Code
     - Swift Code
     - Bank Name
     - Bank Account Number
     - TIN (Tax Identification Number)
   - Fill them with real business values.

15. **Connect `Config` → `Your Company Details`.**

---

## C. Fetch invoices and related clients from NocoDB

16. **Create a `NocoDB` node** named `Get Invoices`.
   - Operation: `Get All`
   - Authentication: `NocoDB API Token`
   - Select your NocoDB credentials
   - Set your Project/Base ID
   - Set the invoices table ID
   - Enable `Return All`

17. **Connect `Your Company Details` → `Get Invoices`.**

18. **Create a `Split In Batches` node** named `Get client info for invoice`.
   - Leave default options unless you want custom batch size.

19. **Connect `Get Invoices` → `Get client info for invoice`.**

20. **Create a `NocoDB` node** named `Get client row`.
   - Authentication: same NocoDB credential
   - Set the clients table ID
   - Set the row ID field to:
     - `={{ $json.Clients_id }}`
   - Use the same Project/Base ID as above.

21. **Connect `Get client info for invoice` second/loop path → `Get client row`.**

22. **Create a `Set` node** named `Add client info to invoice`.
   - Add field `Invoice` of type Object:
     - `={{ $('Get client info for invoice').item.json }}`
   - Add field `Client` of type Object:
     - `={{ $json }}`
   - Do not remove other needed fields unless you want a stricter payload.

23. **Connect `Get client row` → `Add client info to invoice`.**

24. **Connect `Add client info to invoice` back to `Get client info for invoice`** to continue the loop.

Important:
- The table IDs in the original workflow are hardcoded. In a fresh build, use your own NocoDB table IDs.
- This branch assumes the invoice table exposes a client link field that results in `Clients_id`.

---

## D. Filter eligible invoices and classify them into stages

25. **Create a `Filter` node** named `Get only unpaid invoices`.
   - Connect it from the first/main output of `Get client info for invoice`.
   - Recreate the original logic with `OR` conditions:
     1. `Invoice Status` not equal to `Paid`
     2. `Due Date` before `{{$now.minus($('Config').first().json['Days before Due Date for reminder'],'days')}}`
     3. `Payment Date` does not exist
   - Expressions:
     - Left: `={{ $json.Invoice["Invoice Status"] }}`
     - Left date: `={{ Date($json.Invoice["Due Date"]) }}`
     - Left missing value check: `={{ $json.Invoice["Payment Date"] }}``

26. **Create a `Filter` node** named `Get todays / upcoming invoices`.
   - Connect from `Get only unpaid invoices`.
   - Use `OR` conditions:
     1. `={{ $json.Invoice["Due Date"].toDateTime().diffTo($now.format('yyyy-LL-dd'), 'days') }}` equals `={{ $('Config').first().json['Days before Due Date for reminder'] }}`
     2. `={{ $json.Invoice["Due Date"] }}` equals `={{ $now.format('yyyy-LL-dd') }}`

27. **Create a `Filter` node** named `Get overdue invoices for warning`.
   - Connect from `Get only unpaid invoices`.
   - Condition:
     - `={{ $json.Invoice["Due Date"].toDateTime().diffTo($now.format('yyyy-LL-dd'), 'days') }}`
       equals
     - `={{ -$('Config').first().json['Days after Due Date for warning'] }}`

28. **Create a `Filter` node** named `Get overdue invoices for Legal notice`.
   - Connect from `Get only unpaid invoices`.
   - Condition:
     - `={{ $json.Invoice["Due Date"].toDateTime().diffTo($now.format('yyyy-LL-dd'), 'days') }}`
       equals
     - `={{ -$('Config').first().json['Days after Due Date for Legal action'] }}`

Recommended improvement:
- Consider changing the `Get only unpaid invoices` logic from `OR` to `AND` if your intention is to require all unpaid conditions simultaneously.
- Consider using `>=` or range-based overdue logic instead of exact day equality to avoid missed sends when a daily run fails.

---

## E. Build notification content for each stage

29. **Create a `Set` node** named `Prepare Reminder Mail and Letter HTML`.
   - Enable `Include Other Fields`.
   - Add:
     - `Message Content` as String containing full reminder HTML
     - `Email Subject` as String:
       - `=Reminder to pay invoice {{ $json.Invoice["Invoice Id"] }}`
   - In the HTML, reference:
     - invoice data via `$json.Invoice[...]`
     - client data
     - company data via `$('Your Company Details').first().json[...]`
     - config data via `$('Config').first().json[...]`

30. **Connect `Get todays / upcoming invoices` → `Prepare Reminder Mail and Letter HTML`.**

31. **Create a `Set` node** named `Prepare Pre-trial summon Mail and Letter HTML`.
   - Enable `Include Other Fields`.
   - Add:
     - `Message Content` with warning/legal-intent HTML
     - `Email Subject`:
       - `=Will to enter legal route over invoice {{ $json.Invoice["Invoice Id"] }}`

32. **Connect `Get overdue invoices for warning` → `Prepare Pre-trial summon Mail and Letter HTML`.**

33. **Create a `Set` node** named `Prepare Court summon Mail and Letter HTML`.
   - Enable `Include Other Fields`.
   - Add:
     - `Message Content` with legal escalation HTML
     - `Email Subject`:
       - `=Entering Legal route over invoice  {{ $json.Invoice["Invoice Id"] }}`

34. **Connect `Get overdue invoices for Legal notice` → `Prepare Court summon Mail and Letter HTML`.**

35. **Create a `Set` node** named `Combine all inputs`.
   - Enable `Include Other Fields`.
   - Do not add assignments.
   - This node is just a visual convergence point.

36. **Connect all three template nodes into `Combine all inputs`.**

Important note:
- The original HTML uses `$json['Name (from Invoice Clients)']` in some templates. In a fresh rebuild, it is safer to use `{{$json.Client.Name}}` consistently if your combined structure contains `Client`.

---

## F. Add conditional delivery logic

37. **Create an `If` node** named `Send Letter?`.
   - Condition type: Boolean
   - Expression:
     - `={{ $('Config').item.json["Send Physical Letter"] }}`
   - Operation: `true`

38. **Connect `Combine all inputs` → `Send Letter?`.**

39. **Create an `If` node** named `Send Email?`.
   - Condition type: Boolean
   - Expression:
     - `={{ $('Config').item.json["Send Email"] }}`
   - Operation: `true`

40. **Connect `Combine all inputs` → `Send Email?`.**

Safer alternative:
- Use `$('Config').first().json[...]` instead of `.item` if item-linking causes resolution issues in your n8n version.

---

## G. Configure physical letter sending with PostGrid

41. **Create an `HTTP Request` node** named `Send Letter using PostGrid API`.
   - Method: `POST`
   - URL:
     - `https://api.postgrid.com/print-mail/v1/letters`
   - Authentication: `Generic Credential Type`
   - Generic auth type: `HTTP Custom Auth`
   - Body type: JSON
   - Enable `Send Body`

42. **Create PostGrid custom auth credentials**.
   - In Custom Auth, set headers like:
     ```json
     {
       "headers": {
         "x-api-key": "<Your API Key>"
       }
     }
     ```

43. **Set the JSON body** to include:
   - `to` from `$json.Client`
   - `from` from `Your Company Details`
   - `html` from `Message Content` using `JSON.stringify`
   - a description containing invoice ID
   - print options like:
     - `color: false`
     - `doubleSided: false`
     - `addressPlacement: top_first_page`

44. **Connect `Send Letter?` true output → `Send Letter using PostGrid API`.**

---

## H. Configure Gmail email sending

45. **Create a `Gmail` node** named `Send an Email`.
   - Operation: send message
   - Credential: Gmail OAuth2
   - To:
     - `={{ $json.Client.Email }}`
   - Subject:
     - `={{ $json["Email Subject"] }}`
   - Message:
     - `={{ $json["Message Content"] }}`

46. **Create or connect Gmail OAuth2 credentials**.
   - Authorize the Google account allowed to send mail.
   - Ensure the account is permitted for production sending volume.

47. **Connect `Send Email?` true output → `Send an Email`.**

---

## I. Add Slack notification

48. **Create a `Slack` node** named `Send a message`.
   - Authentication: Slack OAuth2
   - Mode: send to channel
   - Choose a channel ID
   - Message type: `block`

49. **Configure Slack block JSON** to include:
   - Subject/message type
   - Invoice number
   - Overdue days
   - Contractor/client name
   - Payment deadline
   - Amount
   - Issue date

50. **Use expressions referencing `Combine all inputs`**, for example:
   - `{{ $('Combine all inputs').item.json['Email Subject'] }}`
   - `{{ $('Combine all inputs').item.json.Invoice["Invoice Id"] }}`
   - `{{ $('Combine all inputs').item.json.Client["Name"] }}`

51. **Connect both delivery nodes to Slack**:
   - `Send an Email` → `Send a message`
   - `Send Letter using PostGrid API` → `Send a message`

Important behavior:
- If both email and letter are enabled, Slack may receive two messages for the same invoice notification.

---

## J. Credentials required

52. Configure these credentials before testing:
- **NocoDB API Token**
  - Used by:
    - `Create NocoDB Invoices Table`
    - `Create NocoDB Clients Table`
    - `Create relation between tables`
    - `Get Invoices`
    - `Get client row`
- **HTTP Custom Auth for PostGrid**
  - Used by:
    - `Send Letter using PostGrid API`
- **Gmail OAuth2**
  - Used by:
    - `Send an Email`
- **Slack OAuth2**
  - Used by:
    - `Send a message`

---

## K. Validation and testing sequence

53. Run the manual setup branch first if you need to create NocoDB tables.

54. Populate NocoDB with sample client and invoice records.
- Ensure invoices include a valid client reference.
- Ensure test due dates match the configured offsets.

55. Test the main branch manually from `Run daily`.

56. Verify:
- invoices are fetched
- client lookup succeeds
- each stage filter catches the expected dates
- HTML is rendered correctly
- PostGrid accepts the address
- Gmail sends correctly
- Slack receives a summary

57. After validation, activate the workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatic vindication of overdue or incoming invoices | General workflow purpose |
| Uses PostGrid for traditional mail | https://www.postgrid.com |
| Uses NocoDB for storing information about invoices | https://nocodb.com |
| Gmail is used for sending emails | Integration used in delivery branch |
| Slack is used for notifying the user about notifications that have been sent | Internal tracking |
| Fill out the `Config` node with all time spans when each type of notification should be sent | Setup note |
| Fill out the `Your Company Details` node because it is used in email/letter templates and PostGrid API calls | Setup note |
| For PostGrid auth, use Custom Auth with header `x-api-key: <Your API Key>` | Delivery setup |
| If you already have `Clients` or `Invoices` tables in NocoDB, you can use different names, but you must edit the JSON bodies in the table-creation nodes and the row ID logic in `Get client row` | Customization note |
| Need help? Contact developers@sailingbyte.com | Project credit / support |

## Additional implementation observations
- The workflow contains **two explicit entry points**:
  - `Run daily` for production execution
  - `Create Tables` for one-time manual schema bootstrap
- There are **no Sub-Workflow / Execute Workflow nodes** in this workflow.
- Some field references in templates mix combined objects (`Client`, `Invoice`) with source-field labels like `Name (from Invoice Clients)`. Standardizing on `Client.Name` is advisable.
- The current filter logic and exact-date matching can lead to over-selection or missed notices if the schedule does not run on the exact expected day.