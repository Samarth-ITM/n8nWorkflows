Record Odoo accounting entries from Telegram using ChatGPT (GPT-4o-mini)

https://n8nworkflows.xyz/workflows/record-odoo-accounting-entries-from-telegram-using-chatgpt--gpt-4o-mini--14496


# Record Odoo accounting entries from Telegram using ChatGPT (GPT-4o-mini)

# 1. Workflow Overview

This workflow turns Arabic chat messages received via webhook into accounting actions in Odoo or reporting responses returned to the chat client. It uses an OpenAI-powered LangChain agent to classify the user’s financial intent, extract structured entities, and then route the request to the proper accounting or reporting branch.

Typical use cases include:

- Recording a direct expense
- Recording a supplier discount
- Recording a supplier payment
- Recording a supplier invoice
- Recording an employee advance
- Asking for a safe/bank/check balance or movement report
- Asking for a supplier statement

The workflow is organized into the following logical blocks.

## 1.1 Input Reception and AI Intent Classification

The workflow starts from a webhook that receives bot/chat payloads. The incoming message body is passed to a LangChain agent using the `gpt-4o-mini` model and short-term memory. The agent is instructed to output only a strict JSON object containing route, action type, company, and extracted financial entities.

## 1.2 AI Output Normalization and Operation Routing

Because LLM output may contain markdown or extra text, a Code node cleans and parses the returned JSON. A Switch node then routes the normalized payload toward one of several accounting or reporting branches based on `route_path`.

## 1.3 Odoo Lookup and Posting Branches

For all “save” operations, the workflow first resolves Odoo internal IDs such as accounts and partners. It then constructs double-entry journal lines in Code nodes, creates draft `account.move` records in Odoo, and immediately posts them by updating their state to `posted`.

This posting logic is split into dedicated branches for:

- Expense entry
- Supplier discount
- Supplier payment by check
- Supplier payment by instant/cash source
- Supplier invoice
- Employee advance

## 1.4 Check Scheduling

If a supplier payment is made via checks (`شيكات`), the workflow formats the due date and creates a Google Calendar reminder event before posting the accounting entry.

## 1.5 Reporting and Chat Response

For “fetch” operations, the workflow either:
- queries Odoo move lines for a safe/bank/check report over a resolved date window, or
- queries Odoo move lines for a supplier ledger summary.

In both cases it builds a formatted Arabic text report and returns it as JSON through `Respond to Webhook`.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and AI Intent Classification

### Overview
This block receives the bot payload and asks the LLM agent to classify the financial intent and extract the accounting entities required by downstream nodes. It is the entry point and semantic parsing layer of the workflow.

### Nodes Involved
- Bot Data Receiver
- AI Financial Agent
- OpenAI Chat Model
- Simple Memory

### Node Details

#### Bot Data Receiver
- **Type and role:** `n8n-nodes-base.webhook`; workflow entry point for inbound HTTP POST requests.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `YOUR_WEBHOOK_UUID`
  - Response mode: `responseNode`, meaning a later `Respond to Webhook` node must send the HTTP response.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - No input; entry node  
  - Outputs to `AI Financial Agent`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or failures:**  
  - Wrong webhook URL or method mismatch
  - Timeout if no downstream `Respond to Webhook` returns a response
  - Payload shape may differ from what the AI prompt expects
- **Sub-workflow reference:** None

#### AI Financial Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LLM-driven classification and entity extraction.
- **Configuration choices:**  
  - Prompt input text is the full JSON-stringified request body: `={{ JSON.stringify($json.body) }}`
  - Prompt type: defined prompt
  - Error handling: `continueRegularOutput`, so the workflow continues even if the agent encounters an issue that still produces output
  - Strong system prompt defines:
    - route decision logic
    - allowed route paths
    - account-mapping placeholders
    - special logic for checks
    - exact output schema
- **Key expressions or variables used:**  
  - `JSON.stringify($json.body)`
  - Expected output fields:
    - `route_path`
    - `company`
    - `action_type`
    - `entities.amount`
    - `entities.source`
    - `entities.target_name`
    - `entities.credit_code`
    - `entities.debit_code`
    - `entities.category_or_type`
    - `entities.date_filter`
    - `entities.check_details`
- **Input and output connections:**  
  - Main input from `Bot Data Receiver`
  - AI language model input from `OpenAI Chat Model`
  - AI memory input from `Simple Memory`
  - Main output to `Clean AI Output`
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain package support in n8n.
- **Edge cases or failures:**  
  - Model may still return non-JSON despite instructions
  - Missing placeholders in prompt may cause invalid accounting mapping
  - If incoming body is not where expected, extraction quality will degrade
  - OpenAI auth, quota, rate limit, or model availability issues
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; provides the actual LLM backend.
- **Configuration choices:**  
  - Model selected from list: `gpt-4o-mini`
  - No additional options configured
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connected to `AI Financial Agent` as `ai_languageModel`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or failures:**  
  - Invalid or missing OpenAI credentials
  - Model access restrictions
  - API latency or throttling
- **Sub-workflow reference:** None

#### Simple Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; stores recent conversational context.
- **Configuration choices:**  
  - Session key: `chat_history`
  - Session ID type: custom key
  - Context window length: `50`
- **Key expressions or variables used:**  
  - Uses `chat_history` as the memory slot
- **Input and output connections:**  
  - Connected to `AI Financial Agent` as `ai_memory`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or failures:**  
  - Without a stable external session identifier, memory continuity may be ineffective
  - May accumulate irrelevant context if webhook callers are not partitioned correctly
- **Sub-workflow reference:** None

---

## 2.2 AI Output Cleaning and Top-Level Routing

### Overview
This block converts the LLM response into valid structured JSON and dispatches the request to the appropriate accounting or reporting branch based on `route_path`.

### Nodes Involved
- Clean AI Output
- Route to Expense Path

### Node Details

#### Clean AI Output
- **Type and role:** `n8n-nodes-base.code`; sanitizes the AI response and parses JSON.
- **Configuration choices:**  
  - Reads first input item
  - Extracts candidate text from `text`, `output`, or `message`
  - Removes markdown code fences
  - Extracts the first `{ ... }` JSON object via regex
  - Parses the cleaned string using `JSON.parse`
- **Key expressions or variables used:**  
  - `inputData.text || inputData.output || inputData.message || ""`
  - Regex cleanup of ```json fences
  - `cleanString.match(/\{[\s\S]*\}/)`
- **Input and output connections:**  
  - Input from `AI Financial Agent`
  - Output to `Route to Expense Path`
- **Version-specific requirements:** Type version `2`
- **Edge cases or failures:**  
  - If no valid JSON exists, `JSON.parse` will throw
  - If the model emits nested braces in surrounding commentary, regex extraction may capture malformed content
  - If AI output schema changes, downstream nodes may fail on missing fields
- **Sub-workflow reference:** None

#### Route to Expense Path
- **Type and role:** `n8n-nodes-base.switch`; routes by `route_path`.
- **Configuration choices:**  
  Outputs are renamed and correspond to:
  - `Expense Path` when `route_path == save_expense`
  - `Discount Path` when `route_path == save_supplier_discount`
  - `Supplier Payment Path` when `route_path == save_supplier_payment`
  - `Supplier Invoice Path` when `route_path == save_supplier_invoice`
  - `Advance Path` when `route_path == save_advance`
  - `Supplier Statement Path` when `route_path == fetch_supplier_statement`
  - `Fetch Path` when `route_path` contains `fetch`
- **Key expressions or variables used:**  
  - `={{ $json.route_path }}`
  - One condition uses `rightValue: =fetch_supplier_statement`, which appears to include a literal leading `=` and may be misconfigured
- **Input and output connections:**  
  - Input from `Clean AI Output`
  - Outputs fan out to:
    - `Find Expense Account ID`
    - `Find Partner ID`
    - `Filter Payment Source`
    - `Find Partner ID4`
    - `Find Partner ID5`
    - `Find Supplier ID`
    - `Date Resolver`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or failures:**  
  - Multiple branches may receive execution if conditions overlap in some Switch configurations; validate behavior in your n8n version
  - The `Supplier Statement Path` condition likely contains an expression formatting issue
  - Unmatched routes produce no output and therefore no webhook response
- **Sub-workflow reference:** None

---

## 2.3 Save Expense Branch

### Overview
This branch records a standard expense by finding the debit and credit accounts in Odoo, creating an `account.move`, posting it, and returning a success message.

### Nodes Involved
- Find Expense Account ID
- Find Cash Account ID
- Build Odoo Journal Entry
- Create Odoo Entry
- Confirm & Post Entry
- Success Message

### Node Details

#### Find Expense Account ID
- **Type and role:** `n8n-nodes-base.odoo`; finds the expense account from the AI-extracted debit code.
- **Configuration choices:**  
  - Resource: custom
  - Operation: `getAll`
  - Custom resource: `account.account`
  - Limit: `1`
  - Filter by `code = entities.debit_code`
- **Key expressions or variables used:**  
  - `={{ $json.entities.debit_code }}`
- **Input and output connections:**  
  - Input from `Route to Expense Path`
  - Output to `Find Cash Account ID`
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:**  
  - No matching account code
  - Multiple matching accounts but only the first is used
  - Odoo auth or connectivity failure
- **Sub-workflow reference:** None

#### Find Cash Account ID
- **Type and role:** `n8n-nodes-base.odoo`; finds the payment source account from AI-extracted credit code.
- **Configuration choices:**  
  - Resource: custom
  - Operation: `getAll`
  - Custom resource: `account.account`
  - Limit: `1`
  - Filter by `code = credit_code`
- **Key expressions or variables used:**  
  - `={{ $('Route to Expense Path').item.json.entities.credit_code }}`
- **Input and output connections:**  
  - Input from `Find Expense Account ID`
  - Output to `Build Odoo Journal Entry`
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:** Same as above
- **Sub-workflow reference:** None

#### Build Odoo Journal Entry
- **Type and role:** `n8n-nodes-base.code`; creates the double-entry payload for Odoo.
- **Configuration choices:**  
  - Pulls amount, category, company, and Odoo account IDs
  - Builds:
    - `ref`
    - `move_type = entry`
    - `journal_id = 2` placeholder
    - `line_ids` as Odoo one2many command array
- **Key expressions or variables used:**  
  - `$('Clean AI Output').item.json`
  - `$('Find Expense Account ID').item.json.id`
  - `$('Find Cash Account ID').item.json.id`
  - Placeholder: `journal_id: 2 // replace`
- **Input and output connections:**  
  - Input from `Find Cash Account ID`
  - Output to `Create Odoo Entry`
- **Version-specific requirements:** Type version `2`
- **Edge cases or failures:**  
  - Invalid amount conversion
  - Missing account IDs
  - Journal ID may not exist in target Odoo instance
- **Sub-workflow reference:** None

#### Create Odoo Entry
- **Type and role:** `n8n-nodes-base.odoo`; creates draft `account.move`.
- **Configuration choices:**  
  - Resource: custom
  - Custom resource: `account.move`
  - Fields set from prior node:
    - `journal_id`
    - `move_type`
    - `line_ids`
    - `ref`
- **Key expressions or variables used:**  
  - `={{ $json.journal_id }}`
  - `={{ $json.move_type }}`
  - `={{ $json.line_ids }}`
  - `={{ $json.ref }}`
- **Input and output connections:**  
  - Input from `Build Odoo Journal Entry`
  - Output to `Confirm & Post Entry`
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:**  
  - Odoo validation errors on journal entry
  - Invalid one2many line format
- **Sub-workflow reference:** None

#### Confirm & Post Entry
- **Type and role:** `n8n-nodes-base.odoo`; updates draft move state to `posted`.
- **Configuration choices:**  
  - Resource: custom
  - Operation: `update`
  - Custom resource: `account.move`
  - Record ID: created move’s `id`
  - Set `state = posted`
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input from `Create Odoo Entry`
  - Output to `Success Message`
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:**  
  - Odoo may not allow direct write to `state` depending on version/customization; some instances require calling a post action method instead
- **Sub-workflow reference:** None

#### Success Message
- **Type and role:** `n8n-nodes-base.code`; generates a generic Arabic success reply.
- **Configuration choices:**  
  - Reads amount and target/category from `Clean AI Output`
  - Builds friendly confirmation text
- **Key expressions or variables used:**  
  - `$('Clean AI Output').first().json.entities`
- **Input and output connections:**  
  - Inputs from all posting confirmation nodes
  - Output to `Respond to Webhook`
- **Version-specific requirements:** Type version `2`
- **Edge cases or failures:**  
  - If `entities.amount` is absent, message may show `0`
- **Sub-workflow reference:** None

---

## 2.4 Save Supplier Discount Branch

### Overview
This branch records a supplier discount or similar partner-linked accounting adjustment by resolving the partner and both accounts, building move lines, creating the move, posting it, and returning success.

### Nodes Involved
- Find Partner ID
- Find Debit Account ID
- Find Credit Account ID
- Build Discount Entry
- Create Discount Entry
- Confirm & Post Entry1
- Success Message

### Node Details

#### Find Partner ID
- **Type and role:** Odoo lookup for supplier/partner.
- **Configuration choices:**  
  - Resource: `res.partner`
  - Operation: `getAll`
  - Limit: `1`
  - Filter `name like entities.target_name`
- **Key expressions or variables used:**  
  - `={{ $json.entities.target_name }}`
- **Input and output connections:**  
  - Input from `Route to Expense Path`
  - Output to `Find Debit Account ID`
- **Edge cases or failures:** fuzzy matching may return wrong partner

#### Find Debit Account ID
- **Type and role:** Odoo lookup for debit-side account.
- **Configuration choices:**  
  - Resource: `account.account`
  - Filter by `code = debit_code`
- **Key expressions or variables used:**  
  - `={{ $('Route to Expense Path').item.json.entities.debit_code }}`
- **Input and output connections:**  
  - From `Find Partner ID` to `Find Credit Account ID`

#### Find Credit Account ID
- **Type and role:** Odoo lookup for credit-side account.
- **Configuration choices:**  
  - Resource: `account.account`
  - Filter by `code = credit_code`
- **Key expressions or variables used:**  
  - `={{ $('Route to Expense Path').item.json.entities.credit_code }}`
- **Input and output connections:**  
  - From `Find Debit Account ID` to `Build Discount Entry`

#### Build Discount Entry
- **Type and role:** Code node to assemble supplier-linked move lines.
- **Configuration choices:**  
  - Uses `partner_id`, debit account, credit account
  - Sets `currencyId = 1` placeholder
  - Uses `amount` and `label` from AI entities
  - Returns `{ lines_to_odoo: lines }`
- **Key expressions or variables used:**  
  - `$('Find Partner ID').item.json.id`
  - `$('Find Debit Account ID').item.json.id`
  - `$('Find Credit Account ID').item.json.id`
  - `$('Clean AI Output').item.json.entities`
- **Input and output connections:**  
  - To `Create Discount Entry`
- **Edge cases or failures:**  
  - `entities.label` is referenced but not defined in the AI schema prompt
  - Invalid currency ID placeholder

#### Create Discount Entry
- **Type and role:** Odoo create `account.move`.
- **Configuration choices:**  
  - Fields:
    - `line_ids = lines_to_odoo`
    - `ref = $('Clean AI Output').item.json.entities.label`
- **Key expressions or variables used:**  
  - `={{ $json.lines_to_odoo }}`
  - `={{ $('Clean AI Output').item.json.entities.label }}`
- **Input and output connections:**  
  - To `Confirm & Post Entry1`
- **Edge cases or failures:**  
  - `ref` may be undefined due to missing `label`

#### Confirm & Post Entry1
- **Type and role:** Odoo update to post move.
- **Configuration choices:** Set `state = posted`
- **Input and output connections:**  
  - To `Success Message`

---

## 2.5 Save Supplier Payment Branch with Source Split

### Overview
This branch handles supplier payments and first decides whether the payment source is checks or an immediate source such as cash/safe/bank. The two subpaths differ mainly in scheduling and line labeling.

### Nodes Involved
- Filter Payment Source
- Format Check Date
- Create an event
- Find Partner ID2
- Find Debit Account ID1
- Find Credit Account ID1
- Build Journal Lines
- Create Discount Entry1
- Confirm & Post Entry2
- Find Partner ID3
- Find Debit Account ID2
- Find Credit Account ID2
- Build Instant Entry Lines
- Create Discount Entry2
- Confirm & Post Entry3
- Success Message

### Node Details

#### Filter Payment Source
- **Type and role:** Switch node to branch based on payment source.
- **Configuration choices:**  
  - `Checks Path` if `entities.source == شيكات`
  - `Instant Payment Path` if `entities.source != شيكات`
- **Key expressions or variables used:**  
  - `={{ $json.entities.source }}`
- **Input and output connections:**  
  - Input from `Route to Expense Path`
  - Outputs to `Format Check Date` and `Find Partner ID3`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or failures:**  
  - Source text must exactly match `شيكات`
  - If AI returns synonyms, checks may not be recognized

### Check payment subpath

#### Format Check Date
- **Type and role:** Code node to convert check due date from `dd/mm/yyyy` to `yyyy-mm-dd`.
- **Configuration choices:**  
  - Iterates all items
  - Reads `entities.check_details.due_date`
  - Stores `entities.check_details.formatted_due_date`
- **Key expressions or variables used:**  
  - `rawDate.split('/')`
- **Input and output connections:**  
  - From `Filter Payment Source`
  - To `Create an event`
- **Edge cases or failures:**  
  - Invalid date format
  - Missing `check_details` or `due_date`

#### Create an event
- **Type and role:** `n8n-nodes-base.googleCalendar`; creates reminder calendar event for due check.
- **Configuration choices:**  
  - Calendar selected from list: `user@example.com`
  - Start at `09:00`, end at `10:00` on formatted due date
  - Summary includes supplier name
  - Description includes amount and check number
  - Default reminders enabled
- **Key expressions or variables used:**  
  - `={{ $json.entities.check_details.formatted_due_date }}T09:00:00`
  - `={{ $json.entities.check_details.formatted_due_date }}T10:00:00`
  - `{{ $json.entities.target_name }}`
  - `{{ $json.entities.amount }}`
  - `{{ $json.entities.check_details.check_number }}`
- **Input and output connections:**  
  - To `Find Partner ID2`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or failures:**  
  - Missing Google Calendar credentials
  - Invalid calendar selection
  - Timezone assumptions may shift event date
- **Sub-workflow reference:** None

#### Find Partner ID2
- **Type and role:** Odoo partner lookup for supplier.
- **Configuration choices:** Filter `name like target_name` from `Format Check Date`
- **Input and output connections:** To `Find Debit Account ID1`

#### Find Debit Account ID1
- **Type and role:** Odoo debit account lookup.
- **Configuration choices:** Filter `code = debit_code`
- **Input and output connections:** To `Find Credit Account ID1`

#### Find Credit Account ID1
- **Type and role:** Odoo credit account lookup.
- **Configuration choices:** Filter `code = credit_code`
- **Input and output connections:** To `Build Journal Lines`

#### Build Journal Lines
- **Type and role:** Code node for check payment journal lines.
- **Configuration choices:**  
  - Uses partner, debit, credit accounts
  - Uses `currencyId = 74` placeholder
  - Builds descriptive line label with check number and due date
  - Returns `journal_ref` and `lines_to_odoo`
- **Key expressions or variables used:**  
  - `$('Format Check Date').item.json.entities`
  - `$('Find Partner ID2').item.json.id`
  - `$('Find Debit Account ID1').item.json.id`
  - `$('Find Credit Account ID1').item.json.id`
- **Input and output connections:** To `Create Discount Entry1`
- **Edge cases or failures:**  
  - Hard-coded currency ID
  - Missing check details

#### Create Discount Entry1
- **Type and role:** Odoo create `account.move` for check payment.
- **Configuration choices:**  
  - Sets `line_ids`
  - Sets `ref = journal_ref`
- **Input and output connections:** To `Confirm & Post Entry2`

#### Confirm & Post Entry2
- **Type and role:** Odoo posting update.
- **Configuration choices:** `state = posted`
- **Input and output connections:** To `Success Message`

### Instant payment subpath

#### Find Partner ID3
- **Type and role:** Odoo partner lookup.
- **Configuration choices:** Filter `name like entities.target_name`
- **Input and output connections:** To `Find Debit Account ID2`

#### Find Debit Account ID2
- **Type and role:** Odoo debit lookup.
- **Configuration choices:** Filter by debit code from `Filter Payment Source`
- **Input and output connections:** To `Find Credit Account ID2`

#### Find Credit Account ID2
- **Type and role:** Odoo credit lookup.
- **Configuration choices:** Filter by credit code from `Filter Payment Source`
- **Input and output connections:** To `Build Instant Entry Lines`

#### Build Instant Entry Lines
- **Type and role:** Code node for immediate supplier payment.
- **Configuration choices:**  
  - Uses current date
  - Uses `currencyId = 74` placeholder
  - Labels entry as immediate supplier payment
  - Returns `journal_ref`, `entry_date`, `lines_to_odoo`
- **Key expressions or variables used:**  
  - `$('Clean AI Output').first().json`
  - `.first().json.id` access on lookup nodes
- **Input and output connections:** To `Create Discount Entry2`
- **Edge cases or failures:**  
  - Current date uses server timezone
  - `entry_date` is returned but never written to Odoo

#### Create Discount Entry2
- **Type and role:** Odoo create `account.move`.
- **Configuration choices:** sets `line_ids` and `ref`
- **Input and output connections:** To `Confirm & Post Entry3`

#### Confirm & Post Entry3
- **Type and role:** Odoo posting update.
- **Input and output connections:** To `Success Message`

---

## 2.6 Save Supplier Invoice Branch

### Overview
This branch records a supplier invoice-style journal entry by resolving partner and accounts, building lines, creating the move, posting it, and returning success.

### Nodes Involved
- Find Partner ID4
- Find Debit Account ID3
- Find Credit Account ID3
- Build Invoice Entry Lines
- Create Discount Entry3
- Confirm & Post Entry4
- Success Message

### Node Details

#### Find Partner ID4
- **Type and role:** Odoo partner lookup with Arabic normalization.
- **Configuration choices:**  
  - Replaces `أإآ` with `ا`
  - Replaces `ى` with `ي`
  - Replaces `ة` with `ه`
  - Trims the result
  - Uses `like` search on partner name
- **Key expressions or variables used:**  
  - `={{ $json.entities.target_name.replace(/[أإآ]/g, 'ا').replace(/[ى]/g, 'ي').replace(/[ة]/g, 'ه').trim() }}`
- **Input and output connections:** To `Find Debit Account ID3`
- **Edge cases or failures:**  
  - Helpful for Arabic normalization, but still fuzzy and may match unintended partners

#### Find Debit Account ID3 / Find Credit Account ID3
- **Type and role:** Odoo account lookups
- **Configuration choices:** Filter by account codes from route payload
- **Input and output connections:** chained into `Build Invoice Entry Lines`

#### Build Invoice Entry Lines
- **Type and role:** Code node to build invoice journal lines.
- **Configuration choices:**  
  - Uses current date
  - Uses `currencyId = 74` placeholder
  - Places partner on the credit line
  - Returns `journal_ref`, `entry_date`, `lines_to_odoo`
- **Key expressions or variables used:**  
  - `$('Clean AI Output').first().json`
  - Lookup node IDs
- **Input and output connections:** To `Create Discount Entry3`

#### Create Discount Entry3
- **Type and role:** Odoo create move
- **Configuration choices:** `line_ids`, `ref`
- **Input and output connections:** To `Confirm & Post Entry4`

#### Confirm & Post Entry4
- **Type and role:** Odoo posting update
- **Input and output connections:** To `Success Message`

---

## 2.7 Save Employee Advance Branch

### Overview
This branch records an employee advance using a partner lookup plus debit and credit account resolution, then creates and posts the journal entry.

### Nodes Involved
- Find Partner ID5
- Find Debit Account ID4
- Find Credit Account ID4
- Build Advance Entry Lines
- Create Discount Entry4
- Confirm & Post Entry5
- Success Message

### Node Details

#### Find Partner ID5
- **Type and role:** Odoo partner lookup with Arabic normalization.
- **Configuration choices:** same normalization strategy as `Find Partner ID4`
- **Input and output connections:** To `Find Debit Account ID4`

#### Find Debit Account ID4 / Find Credit Account ID4
- **Type and role:** Odoo account lookups for advance posting.
- **Configuration choices:** filter by AI account codes
- **Input and output connections:** chained into `Build Advance Entry Lines`

#### Build Advance Entry Lines
- **Type and role:** Code node to construct employee advance move lines.
- **Configuration choices:**  
  - Uses current date
  - Uses `currencyId = 74` placeholder
  - Sets partner on debit line
  - Returns `journal_ref`, `entry_date`, `lines_to_odoo`
- **Key expressions or variables used:**  
  - `$('Clean AI Output').first().json`
  - lookup IDs
- **Input and output connections:** To `Create Discount Entry4`

#### Create Discount Entry4
- **Type and role:** Odoo create move
- **Configuration choices:** `line_ids`, `ref`
- **Input and output connections:** To `Confirm & Post Entry5`

#### Confirm & Post Entry5
- **Type and role:** Odoo posting update
- **Input and output connections:** To `Success Message`

---

## 2.8 Fetch Safe/Bank/Expense Report Branch

### Overview
This branch handles fetch-style account reporting by resolving a date window from the user message, querying Odoo move lines, summarizing debits and credits, and returning a formatted Arabic report.

### Nodes Involved
- Date Resolver
- Fetch Safe/Bank Move Lines
- Build Final Report
- Respond to Webhook

### Node Details

#### Date Resolver
- **Type and role:** Code node to infer reporting period from text.
- **Configuration choices:**  
  - Reads `user_message`
  - If message contains `شهر` or `الشهر`, uses current month start to today
  - If message contains `امبارح`, uses yesterday
  - Else defaults to today
  - Returns `startDate`, `endDate`, `display_name`
- **Key expressions or variables used:**  
  - `const userMessage = input.user_message || ""`
- **Input and output connections:**  
  - Input from `Route to Expense Path`
  - Output to `Fetch Safe/Bank Move Lines`
- **Edge cases or failures:**  
  - Upstream payload may not contain `user_message`; branch then defaults to today
  - `endDate` is computed but not used downstream

#### Fetch Safe/Bank Move Lines
- **Type and role:** Odoo query against `account.move.line`.
- **Configuration choices:**  
  - Resource: custom
  - Operation: `getAll`
  - Custom resource: `account.move.line`
  - Limit set to `0` via expression
  - Filter by:
    - `account_code = credit_code`
    - `date = startDate`
  - `alwaysOutputData = true`
- **Key expressions or variables used:**  
  - `={{ $('Route to Expense Path').item.json.entities.credit_code }}`
  - `={{ $json.startDate }}`
- **Input and output connections:**  
  - From `Date Resolver`
  - To `Build Final Report`
- **Edge cases or failures:**  
  - Month reporting is incomplete because only exact `startDate` is queried; no range filter is used
  - If Odoo expects a different field name than `account_code`, query may fail or return nothing
  - `limit = 0` behavior depends on node implementation

#### Build Final Report
- **Type and role:** Code node to create Arabic text report.
- **Configuration choices:**  
  - Aggregates debits and credits over returned items
  - Attempts to infer account name from move line `account_id[1]`
  - Switches output style depending on whether account appears to be checks/cash/safe
  - Falls back to “no movements” message if there are no lines
- **Key expressions or variables used:**  
  - `$('Clean AI Output').first().json.entities`
  - `$('Date Resolver').first().json`
- **Input and output connections:**  
  - To `Respond to Webhook`
- **Edge cases or failures:**  
  - Account naming heuristics are text-based and language-dependent
  - No date-range handling despite resolver supporting month/yesterday/today labels

#### Respond to Webhook
- **Type and role:** `n8n-nodes-base.respondToWebhook`; returns final HTTP JSON response.
- **Configuration choices:**  
  - Respond with JSON
  - Body: `{ "reply": $json.report_text }`
- **Key expressions or variables used:**  
  - `={{ { "reply": $json.report_text } }}`
- **Input and output connections:**  
  - Receives from `Build Final Report`, `Calculate Supplier Balance`, and `Success Message`
- **Version-specific requirements:** Type version `1.5`
- **Edge cases or failures:**  
  - If branch never reaches this node, webhook caller hangs until timeout
- **Sub-workflow reference:** None

---

## 2.9 Fetch Supplier Statement Branch

### Overview
This branch locates the supplier, retrieves all posted move lines linked to that partner, computes a running aggregate, and returns a concise Arabic supplier statement.

### Nodes Involved
- Find Supplier ID
- Get many items
- Calculate Supplier Balance
- Respond to Webhook

### Node Details

#### Find Supplier ID
- **Type and role:** Odoo partner lookup with Arabic normalization.
- **Configuration choices:**  
  - Searches `res.partner`
  - Limit `1`
  - `name like normalized target_name`
- **Key expressions or variables used:** same normalization expression used in other partner lookups
- **Input and output connections:**  
  - From `Route to Expense Path`
  - To `Get many items`
- **Edge cases or failures:**  
  - Wrong partner due to fuzzy match

#### Get many items
- **Type and role:** Odoo ledger query for partner move lines.
- **Configuration choices:**  
  - Resource: custom
  - Operation: `getAll`
  - Return all: `true`
  - Custom resource: `account.move.line`
  - Filters:
    - `partner_id = found supplier id`
    - `parent_state = posted`
  - `alwaysOutputData = true`
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - To `Calculate Supplier Balance`
- **Edge cases or failures:**  
  - Large ledger could return many records and affect performance

#### Calculate Supplier Balance
- **Type and role:** Code node to summarize supplier balance and recent activity.
- **Configuration choices:**  
  - Sorts ledger lines descending by date
  - Computes total debit and total credit
  - Builds up to 10 latest movement lines
  - Computes net as `totalCredit - totalDebit`
  - Outputs Arabic report text indicating whether you owe the supplier or vice versa
- **Key expressions or variables used:**  
  - `$('Clean AI Output').first().json.entities.target_name`
- **Input and output connections:**  
  - To `Respond to Webhook`
- **Edge cases or failures:**  
  - Interpretation of debit/credit depends on accounting conventions and selected move lines
  - No pagination for very large datasets before processing

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Bot Data Receiver | Webhook | Receives inbound bot/chat POST requests |  | AI Financial Agent | ## 1. Receive & Classify Intent<br>Receives the chat message and uses an AI agent to extract the financial operation, amounts, dates, and target accounts into a clean JSON format. |
| OpenAI Chat Model | OpenAI Chat Model | Provides GPT-4o-mini model to the LangChain agent |  | AI Financial Agent | ## 1. Receive & Classify Intent<br>Receives the chat message and uses an AI agent to extract the financial operation, amounts, dates, and target accounts into a clean JSON format. |
| Simple Memory | Memory Buffer Window | Supplies short-term conversational context to the agent |  | AI Financial Agent | ## 1. Receive & Classify Intent<br>Receives the chat message and uses an AI agent to extract the financial operation, amounts, dates, and target accounts into a clean JSON format. |
| AI Financial Agent | LangChain Agent | Classifies financial intent and extracts structured JSON | Bot Data Receiver, OpenAI Chat Model, Simple Memory | Clean AI Output | ## 1. Receive & Classify Intent<br>Receives the chat message and uses an AI agent to extract the financial operation, amounts, dates, and target accounts into a clean JSON format. |
| Clean AI Output | Code | Cleans markdown-wrapped LLM output and parses JSON | AI Financial Agent | Route to Expense Path | ## 1. Receive & Classify Intent<br>Receives the chat message and uses an AI agent to extract the financial operation, amounts, dates, and target accounts into a clean JSON format. |
| Route to Expense Path | Switch | Routes the request to save/fetch branches by route_path | Clean AI Output | Find Expense Account ID, Find Partner ID, Filter Payment Source, Find Partner ID4, Find Partner ID5, Find Supplier ID, Date Resolver | ## 2. Route Operation<br>Directs the workflow down the correct path based on whether the user is logging an expense, paying a supplier, or requesting a balance report. |
| Find Expense Account ID | Odoo | Finds expense/debit account for direct expense posting | Route to Expense Path | Find Cash Account ID | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Cash Account ID | Odoo | Finds source/cash account for direct expense posting | Find Expense Account ID | Build Odoo Journal Entry | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Partner ID | Odoo | Finds supplier/partner for discount branch | Route to Expense Path | Find Debit Account ID | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Debit Account ID | Odoo | Finds debit account for discount branch | Find Partner ID | Find Credit Account ID | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Credit Account ID | Odoo | Finds credit account for discount branch | Find Debit Account ID | Build Discount Entry | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Partner ID2 | Odoo | Finds supplier for check payment branch | Create an event | Find Debit Account ID1 | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Debit Account ID1 | Odoo | Finds debit account for check payment branch | Find Partner ID2 | Find Credit Account ID1 | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Credit Account ID1 | Odoo | Finds credit account for check payment branch | Find Debit Account ID1 | Build Journal Lines | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Partner ID3 | Odoo | Finds supplier for instant payment branch | Filter Payment Source | Find Debit Account ID2 | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Debit Account ID2 | Odoo | Finds debit account for instant payment branch | Find Partner ID3 | Find Credit Account ID2 | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Credit Account ID2 | Odoo | Finds credit account for instant payment branch | Find Debit Account ID2 | Build Instant Entry Lines | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Partner ID4 | Odoo | Finds supplier for invoice branch with Arabic normalization | Route to Expense Path | Find Debit Account ID3 | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Debit Account ID3 | Odoo | Finds debit account for supplier invoice branch | Find Partner ID4 | Find Credit Account ID3 | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Credit Account ID3 | Odoo | Finds credit account for supplier invoice branch | Find Debit Account ID3 | Build Invoice Entry Lines | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Partner ID5 | Odoo | Finds partner for employee advance branch with Arabic normalization | Route to Expense Path | Find Debit Account ID4 | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Debit Account ID4 | Odoo | Finds debit account for advance branch | Find Partner ID5 | Find Credit Account ID4 | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Credit Account ID4 | Odoo | Finds credit account for advance branch | Find Debit Account ID4 | Build Advance Entry Lines | ## 3. Odoo Data Lookups<br>Searches Odoo to find the internal internal `account_id` and `partner_id` required to build accurate journal entries. |
| Find Supplier ID | Odoo | Finds supplier for statement/report branch | Route to Expense Path | Get many items | ## 6. Reports & Chat Response<br>Calculates net balances from Odoo ledger lines, generates a formatted Arabic summary, and sends the response back to the user via Webhook. |
| Get many items | Odoo | Retrieves posted move lines for supplier statement | Find Supplier ID | Calculate Supplier Balance | ## 6. Reports & Chat Response<br>Calculates net balances from Odoo ledger lines, generates a formatted Arabic summary, and sends the response back to the user via Webhook. |
| Calculate Supplier Balance | Code | Builds Arabic supplier statement summary | Get many items | Respond to Webhook | ## 6. Reports & Chat Response<br>Calculates net balances from Odoo ledger lines, generates a formatted Arabic summary, and sends the response back to the user via Webhook. |
| Date Resolver | Code | Resolves reporting dates from Arabic message text | Route to Expense Path | Fetch Safe/Bank Move Lines | ## 6. Reports & Chat Response<br>Calculates net balances from Odoo ledger lines, generates a formatted Arabic summary, and sends the response back to the user via Webhook. |
| Fetch Safe/Bank Move Lines | Odoo | Retrieves move lines for cash/safe/bank report | Date Resolver | Build Final Report | ## 6. Reports & Chat Response<br>Calculates net balances from Odoo ledger lines, generates a formatted Arabic summary, and sends the response back to the user via Webhook. |
| Build Final Report | Code | Builds Arabic account movement report | Fetch Safe/Bank Move Lines | Respond to Webhook | ## 6. Reports & Chat Response<br>Calculates net balances from Odoo ledger lines, generates a formatted Arabic summary, and sends the response back to the user via Webhook. |
| Respond to Webhook | Respond to Webhook | Returns the final reply JSON to the caller | Build Final Report, Calculate Supplier Balance, Success Message |  | ## 6. Reports & Chat Response<br>Calculates net balances from Odoo ledger lines, generates a formatted Arabic summary, and sends the response back to the user via Webhook. |
| Build Odoo Journal Entry | Code | Builds direct expense account.move payload | Find Cash Account ID | Create Odoo Entry | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Create Odoo Entry | Odoo | Creates draft direct expense journal entry | Build Odoo Journal Entry | Confirm & Post Entry | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Confirm & Post Entry | Odoo | Posts direct expense journal entry | Create Odoo Entry | Success Message | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Build Discount Entry | Code | Builds supplier discount move lines | Find Credit Account ID | Create Discount Entry | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Create Discount Entry | Odoo | Creates supplier discount move | Build Discount Entry | Confirm & Post Entry1 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Confirm & Post Entry1 | Odoo | Posts supplier discount move | Create Discount Entry | Success Message | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Build Journal Lines | Code | Builds supplier payment by check move lines | Find Credit Account ID1 | Create Discount Entry1 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Create Discount Entry1 | Odoo | Creates supplier payment by check move | Build Journal Lines | Confirm & Post Entry2 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Confirm & Post Entry2 | Odoo | Posts supplier payment by check move | Create Discount Entry1 | Success Message | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Build Instant Entry Lines | Code | Builds immediate supplier payment move lines | Find Credit Account ID2 | Create Discount Entry2 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Create Discount Entry2 | Odoo | Creates immediate supplier payment move | Build Instant Entry Lines | Confirm & Post Entry3 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Confirm & Post Entry3 | Odoo | Posts immediate supplier payment move | Create Discount Entry2 | Success Message | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Build Invoice Entry Lines | Code | Builds supplier invoice move lines | Find Credit Account ID3 | Create Discount Entry3 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Create Discount Entry3 | Odoo | Creates supplier invoice move | Build Invoice Entry Lines | Confirm & Post Entry4 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Confirm & Post Entry4 | Odoo | Posts supplier invoice move | Create Discount Entry3 | Success Message | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Build Advance Entry Lines | Code | Builds employee advance move lines | Find Credit Account ID4 | Create Discount Entry4 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Create Discount Entry4 | Odoo | Creates employee advance move | Build Advance Entry Lines | Confirm & Post Entry5 | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Confirm & Post Entry5 | Odoo | Posts employee advance move | Create Discount Entry4 | Success Message | ## 4. Build & Post Entries<br>Constructs the double-entry accounting arrays (debits and credits) and automatically posts the confirmed journal entries into Odoo. |
| Filter Payment Source | Switch | Splits supplier payment into check vs non-check paths | Route to Expense Path | Format Check Date, Find Partner ID3 | ## 5. Schedule Checks<br>Parses future due dates for physical check payments and creates a Google Calendar event reminder. |
| Format Check Date | Code | Formats check due date for calendar/event use | Filter Payment Source | Create an event | ## 5. Schedule Checks<br>Parses future due dates for physical check payments and creates a Google Calendar event reminder. |
| Create an event | Google Calendar | Creates due-date reminder for check payments | Format Check Date | Find Partner ID2 | ## 5. Schedule Checks<br>Parses future due dates for physical check payments and creates a Google Calendar event reminder. |
| Success Message | Code | Generates final success reply after posting | Confirm & Post Entry, Confirm & Post Entry1, Confirm & Post Entry2, Confirm & Post Entry3, Confirm & Post Entry4, Confirm & Post Entry5 | Respond to Webhook |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `Automate Odoo Accounting with Telegram and ChatGPT`.

2. **Add a Webhook node** named `Bot Data Receiver`.
   - Method: `POST`
   - Path: choose a unique value such as `YOUR_WEBHOOK_UUID`
   - Response mode: `Using Respond to Webhook node`
   - This will be the main external entry point for Telegram, Botpress, or another bot platform.

3. **Add an OpenAI Chat Model node** named `OpenAI Chat Model`.
   - Provider credentials: OpenAI API
   - Model: `gpt-4o-mini`

4. **Add a Memory Buffer Window node** named `Simple Memory`.
   - Session key: `chat_history`
   - Session ID type: `Custom Key`
   - Context window length: `50`

5. **Add an AI Agent node** named `AI Financial Agent`.
   - Connect:
     - `Bot Data Receiver` main output -> `AI Financial Agent` main input
     - `OpenAI Chat Model` -> agent language model input
     - `Simple Memory` -> agent memory input
   - Set prompt text to: `={{ JSON.stringify($json.body) }}`
   - Set a system message instructing the model to:
     - output only JSON
     - classify route path
     - extract amount, company, source, target name, debit code, credit code, category, date filters, check details
     - follow your actual account mapping
   - Replace all placeholders such as:
     - `[SAFE_1_CODE]`
     - `[EXPENSE_1_CODE]`
     - `[COMPANY_A]`
     - `[CHECKS_CODE]`

6. **Add a Code node** named `Clean AI Output`.
   - Paste logic that:
     - reads `text`, `output`, or `message`
     - strips markdown fences
     - extracts the JSON object
     - parses it with `JSON.parse`
   - Connect `AI Financial Agent` -> `Clean AI Output`

7. **Add a Switch node** named `Route to Expense Path`.
   - Connect `Clean AI Output` -> `Route to Expense Path`
   - Add renamed outputs for:
     1. `Expense Path` when `route_path = save_expense`
     2. `Discount Path` when `route_path = save_supplier_discount`
     3. `Supplier Payment Path` when `route_path = save_supplier_payment`
     4. `Supplier Invoice Path` when `route_path = save_supplier_invoice`
     5. `Advance Path` when `route_path = save_advance`
     6. `Supplier Statement Path` when `route_path = fetch_supplier_statement`
     7. `Fetch Path` when `route_path contains fetch`
   - Ensure the `fetch_supplier_statement` comparison does not accidentally include a leading literal `=`.

---

## Build the direct expense branch

8. **Add an Odoo node** named `Find Expense Account ID`.
   - Credentials: Odoo
   - Resource: `Custom`
   - Operation: `Get All`
   - Custom resource: `account.account`
   - Limit: `1`
   - Filter: `code = {{$json.entities.debit_code}}`

9. **Add an Odoo node** named `Find Cash Account ID`.
   - Resource: `account.account`
   - Filter: `code = {{ $('Route to Expense Path').item.json.entities.credit_code }}`
   - Limit: `1`

10. **Add a Code node** named `Build Odoo Journal Entry`.
    - Build an object with:
      - `move_type: "entry"`
      - `journal_id`: your Odoo journal internal ID
      - `ref`: category + company
      - `line_ids`: two lines, debit and credit
    - Replace hard-coded `journal_id = 2` with your real journal ID.

11. **Add an Odoo node** named `Create Odoo Entry`.
    - Resource: `account.move`
    - Create fields:
      - `journal_id`
      - `move_type`
      - `line_ids`
      - `ref`

12. **Add an Odoo node** named `Confirm & Post Entry`.
    - Operation: `Update`
    - Resource: `account.move`
    - Record ID: `={{ $json.id }}`
    - Set field `state = posted`

13. Connect the branch:
    - `Route to Expense Path` -> `Find Expense Account ID`
    - `Find Expense Account ID` -> `Find Cash Account ID`
    - `Find Cash Account ID` -> `Build Odoo Journal Entry`
    - `Build Odoo Journal Entry` -> `Create Odoo Entry`
    - `Create Odoo Entry` -> `Confirm & Post Entry`

---

## Build the supplier discount branch

14. **Add an Odoo node** named `Find Partner ID`.
    - Resource: `res.partner`
    - Operation: `Get All`
    - Filter: `name like {{$json.entities.target_name}}`
    - Limit: `1`

15. **Add Odoo nodes** named `Find Debit Account ID` and `Find Credit Account ID`.
    - Both query `account.account`
    - Debit filter: route payload `entities.debit_code`
    - Credit filter: route payload `entities.credit_code`
    - Limit each to `1`

16. **Add a Code node** named `Build Discount Entry`.
    - Use:
      - partner ID
      - debit account ID
      - credit account ID
      - amount from AI
      - label from AI
    - Return `{ lines_to_odoo: [...] }`
    - Replace placeholder `currencyId = 1` with your actual Odoo currency ID
    - Strongly consider adding `label` to the AI schema, since the current prompt does not define it.

17. **Add an Odoo node** named `Create Discount Entry`.
    - Resource: `account.move`
    - Fields:
      - `line_ids = {{$json.lines_to_odoo}}`
      - `ref = {{ $('Clean AI Output').item.json.entities.label }}`

18. **Add an Odoo node** named `Confirm & Post Entry1`.
    - Update `account.move`
    - Record ID: `={{ $json.id }}`
    - Set `state = posted`

19. Connect:
    - `Route to Expense Path` -> `Find Partner ID`
    - `Find Partner ID` -> `Find Debit Account ID`
    - `Find Debit Account ID` -> `Find Credit Account ID`
    - `Find Credit Account ID` -> `Build Discount Entry`
    - `Build Discount Entry` -> `Create Discount Entry`
    - `Create Discount Entry` -> `Confirm & Post Entry1`

---

## Build the supplier payment split

20. **Add a Switch node** named `Filter Payment Source`.
    - Input from `Route to Expense Path`
    - Output 1: `Checks Path` when `entities.source = شيكات`
    - Output 2: `Instant Payment Path` when `entities.source != شيكات`

### Check payment subpath

21. **Add a Code node** named `Format Check Date`.
    - Convert `dd/mm/yyyy` into `yyyy-mm-dd`
    - Save it as `entities.check_details.formatted_due_date`

22. **Add a Google Calendar node** named `Create an event`.
    - Credentials: Google Calendar OAuth2
    - Calendar: choose the target calendar
    - Start: `={{ $json.entities.check_details.formatted_due_date }}T09:00:00`
    - End: `={{ $json.entities.check_details.formatted_due_date }}T10:00:00`
    - Summary: supplier + due check text
    - Description: amount + check number
    - Enable default reminders

23. **Add Odoo lookup nodes**:
    - `Find Partner ID2`
    - `Find Debit Account ID1`
    - `Find Credit Account ID1`

24. **Add a Code node** named `Build Journal Lines`.
    - Use:
      - partner
      - debit and credit accounts
      - amount
      - check number
      - formatted due date
    - Return:
      - `journal_ref`
      - `lines_to_odoo`
    - Replace `currencyId = 74` with your real Odoo currency ID.

25. **Add Odoo node** `Create Discount Entry1`.
    - Create `account.move`
    - Fields:
      - `line_ids`
      - `ref = journal_ref`

26. **Add Odoo node** `Confirm & Post Entry2`.
    - Update move by `id`
    - Set `state = posted`

27. Connect:
    - `Filter Payment Source` -> `Format Check Date`
    - `Format Check Date` -> `Create an event`
    - `Create an event` -> `Find Partner ID2`
    - `Find Partner ID2` -> `Find Debit Account ID1`
    - `Find Debit Account ID1` -> `Find Credit Account ID1`
    - `Find Credit Account ID1` -> `Build Journal Lines`
    - `Build Journal Lines` -> `Create Discount Entry1`
    - `Create Discount Entry1` -> `Confirm & Post Entry2`

### Instant payment subpath

28. **Add Odoo lookup nodes**:
    - `Find Partner ID3`
    - `Find Debit Account ID2`
    - `Find Credit Account ID2`

29. **Add a Code node** named `Build Instant Entry Lines`.
    - Use current date
    - Build debit/credit lines for immediate payment
    - Return:
      - `journal_ref`
      - `entry_date`
      - `lines_to_odoo`
    - Replace `currencyId = 74` with your real ID

30. **Add Odoo node** `Create Discount Entry2`.
    - Create `account.move`
    - Set `line_ids` and `ref`

31. **Add Odoo node** `Confirm & Post Entry3`.
    - Update move state to `posted`

32. Connect:
    - `Filter Payment Source` -> `Find Partner ID3`
    - `Find Partner ID3` -> `Find Debit Account ID2`
    - `Find Debit Account ID2` -> `Find Credit Account ID2`
    - `Find Credit Account ID2` -> `Build Instant Entry Lines`
    - `Build Instant Entry Lines` -> `Create Discount Entry2`
    - `Create Discount Entry2` -> `Confirm & Post Entry3`

---

## Build the supplier invoice branch

33. **Add Odoo node** `Find Partner ID4`.
    - Use normalized Arabic name expression before `like` matching

34. **Add Odoo nodes**:
    - `Find Debit Account ID3`
    - `Find Credit Account ID3`

35. **Add Code node** `Build Invoice Entry Lines`.
    - Build invoice-style double entry
    - Replace `currencyId = 74`

36. **Add Odoo node** `Create Discount Entry3`

37. **Add Odoo node** `Confirm & Post Entry4`

38. Connect the branch from `Route to Expense Path` through those nodes in order.

---

## Build the employee advance branch

39. **Add Odoo node** `Find Partner ID5`.
    - Use the same Arabic normalization pattern

40. **Add Odoo nodes**:
    - `Find Debit Account ID4`
    - `Find Credit Account ID4`

41. **Add Code node** `Build Advance Entry Lines`.
    - Build employee advance lines
    - Replace `currencyId = 74`

42. **Add Odoo node** `Create Discount Entry4`

43. **Add Odoo node** `Confirm & Post Entry5`

44. Connect the branch from `Route to Expense Path` through those nodes in order.

---

## Build the reporting branches

### Safe/bank/check report branch

45. **Add a Code node** named `Date Resolver`.
    - Read the user text
    - Resolve:
      - current month
      - yesterday
      - today
    - Output:
      - `startDate`
      - `endDate`
      - `display_name`
    - If your inbound payload does not include `user_message`, adapt the node to read the actual field from your webhook body.

46. **Add an Odoo node** named `Fetch Safe/Bank Move Lines`.
    - Resource: `account.move.line`
    - Operation: `Get All`
    - Filter:
      - `account_code = route payload credit_code`
      - `date = startDate`
    - Turn on `Always Output Data`
    - If you need monthly reports, replace exact date filtering with a proper date range query.

47. **Add a Code node** named `Build Final Report`.
    - Aggregate debit and credit totals
    - Build a formatted Arabic report string
    - Return `{ report_text }`

48. Connect:
    - `Route to Expense Path` -> `Date Resolver`
    - `Date Resolver` -> `Fetch Safe/Bank Move Lines`
    - `Fetch Safe/Bank Move Lines` -> `Build Final Report`

### Supplier statement branch

49. **Add an Odoo node** named `Find Supplier ID`.
    - Lookup `res.partner` using normalized Arabic name

50. **Add an Odoo node** named `Get many items`.
    - Resource: `account.move.line`
    - Operation: `Get All`
    - Return all: enabled
    - Filters:
      - `partner_id = found supplier id`
      - `parent_state = posted`
    - Turn on `Always Output Data`

51. **Add a Code node** named `Calculate Supplier Balance`.
    - Sort move lines by date descending
    - Build the latest 10 movement lines
    - Compute:
      - total debit
      - total credit
      - balance = totalCredit - totalDebit
    - Return `{ report_text }`

52. Connect:
    - `Route to Expense Path` -> `Find Supplier ID`
    - `Find Supplier ID` -> `Get many items`
    - `Get many items` -> `Calculate Supplier Balance`

---

## Add shared response handling

53. **Add a Code node** named `Success Message`.
    - Build an Arabic success confirmation using:
      - `amount`
      - `target_name` or `category_or_type`

54. Connect all posting nodes to `Success Message`:
    - `Confirm & Post Entry`
    - `Confirm & Post Entry1`
    - `Confirm & Post Entry2`
    - `Confirm & Post Entry3`
    - `Confirm & Post Entry4`
    - `Confirm & Post Entry5`

55. **Add a Respond to Webhook node** named `Respond to Webhook`.
    - Respond with JSON
    - Response body: `={{ { "reply": $json.report_text } }}`

56. Connect:
    - `Success Message` -> `Respond to Webhook`
    - `Build Final Report` -> `Respond to Webhook`
    - `Calculate Supplier Balance` -> `Respond to Webhook`

---

## Credentials required

57. **Configure Odoo credentials**
    - Base URL
    - Database name
    - Username/email
    - Password or API-compatible auth depending on your n8n/Odoo setup

58. **Configure OpenAI credentials**
    - API key with access to `gpt-4o-mini`

59. **Configure Google Calendar credentials**
    - OAuth2
    - Select the calendar used for check reminders

---

## Important adjustments before production use

60. Replace all placeholders in prompts and Code nodes:
    - account codes
    - company names
    - journal IDs
    - currency IDs
    - calendar value
    - webhook path

61. Fix schema mismatches:
    - `entities.label` is used in discount nodes but is not defined in the AI system prompt
    - either add it to the prompt schema or change those nodes to use another field

62. Fix route condition issues:
    - verify the `fetch_supplier_statement` switch rule does not include an erroneous literal `=`

63. Improve fetch date logic if needed:
    - current monthly logic calculates a range but queries only one exact date

64. Test all branches independently with sample Arabic payloads:
    - direct expense
    - supplier discount
    - supplier payment with `شيكات`
    - supplier payment without checks
    - supplier invoice
    - advance
    - balance report
    - supplier statement

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow acts as an AI-powered financial assistant, connecting chat platforms (like Telegram or Botpress) to your Odoo ERP. When a user sends a natural language message in Arabic (e.g., "I spent 2000 on office supplies"), an OpenAI LangChain agent extracts the intent, amounts, dates, and account categories. The workflow then routes the request to either post a new double-entry journal record in Odoo (for expenses, supplier payments, or employee advances) or fetch real-time account balances to send a formatted Arabic report back to the chat. | General workflow note |
| Setup steps: 1. Connect Odoo, OpenAI, and Google Calendar credentials. 2. Add the Webhook URL to your Telegram/Botpress bot setup. 3. Open the "AI Financial Agent" node and update the `# ACCOUNT MAPPING` section with your specific Odoo Chart of Accounts codes. 4. Open the "Build [X] Entry" Code nodes and replace placeholder `journal_id` and `currency_id` values with your Odoo internal IDs. | General setup guidance |
| Customization tip: You can expand the `Route to Expense Path` Switch node to handle additional operations, such as creating customer invoices or logging petty cash transfers. | Extension idea |

## Additional implementation observations
| Note Content | Context or Link |
|---|---|
| The workflow has a single technical entry point: `Bot Data Receiver` webhook. | Entry-point note |
| There are no Execute Workflow or sub-workflow nodes in this workflow. | Sub-workflow note |
| Several nodes named `Create Discount EntryX` are reused for different transaction types; renaming them by business purpose would improve maintainability. | Maintainability note |
| Multiple posting nodes directly write `state = posted` on `account.move`. In some Odoo environments, posting may require calling a dedicated action instead of a field update. | Odoo compatibility note |
| Partner matching relies on `like` filters and partial Arabic normalization. This is useful but may produce incorrect matches in large partner datasets. | Data quality note |
| The reporting branch currently computes a date range but queries move lines only for `startDate`, not for the whole range. | Logic caveat |