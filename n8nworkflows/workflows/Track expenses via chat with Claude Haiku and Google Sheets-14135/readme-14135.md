Track expenses via chat with Claude Haiku and Google Sheets

https://n8nworkflows.xyz/workflows/track-expenses-via-chat-with-claude-haiku-and-google-sheets-14135


# Track expenses via chat with Claude Haiku and Google Sheets

# 1. Workflow Overview

This workflow is a chat-based expense tracker built in n8n. A user sends a natural-language message such as “spent 500 on lunch” or “summary february,” and the workflow either:

- extracts and stores a new expense in Google Sheets,
- generates a monthly spending summary from the sheet,
- or returns a help message.

It uses an Anthropic Claude Haiku model to parse free-form expense messages into structured fields, then persists them to a Google Sheet with a running monthly total.

## 1.1 Entry and Intent Detection

The workflow starts from a chat trigger, reads the incoming message, and classifies it into one of three intents:

- `expense`
- `summary`
- `help`

This routing determines which downstream branch runs.

## 1.2 Expense Parsing and Storage

If the message is treated as an expense, the workflow loads all existing expense rows from Google Sheets, prepares context, sends the raw message to Claude Haiku for structured extraction, validates the AI result, computes the correct month and running total, and appends the new expense to the sheet.

## 1.3 Summary Generation

If the message is a summary request, the workflow reads all rows from the same Google Sheet, determines the requested month from the chat text, aggregates expenses by category, and returns a formatted monthly report.

## 1.4 Help Response

If the user asks for help, the workflow returns usage instructions, supported command formats, currencies, and category behavior.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Chat Entry

### Overview
This block is the workflow’s single entry point. It receives each chat message and forwards the payload into the intent classifier.

### Nodes Involved
- When chat message received

### Node Details

#### When chat message received
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Entry trigger for chat-based workflows.
- **Configuration choices:**  
  Uses default options. It waits for inbound chat messages and exposes message fields such as `chatInput` or sometimes `input`.
- **Key expressions or variables used:**  
  No direct expressions configured here, but downstream nodes read:
  - `$input.first().json.chatInput`
  - `$input.first().json.input`
- **Input and output connections:**  
  - Input: none, trigger node
  - Output: `Detect Intent`
- **Version-specific requirements:**  
  `typeVersion: 1.1`
- **Edge cases or potential failure types:**  
  - Different chat frontends may provide text under different keys; downstream code compensates by checking both `chatInput` and `input`.
  - If the trigger payload is empty, downstream intent detection will classify an empty string.
- **Sub-workflow reference:**  
  None

---

## 2.2 Block: Intent Detection and Routing

### Overview
This block normalizes the incoming message and classifies it into one of three intents. A Switch node then sends execution to the expense, summary, or help branch.

### Nodes Involved
- Detect Intent
- Intent Switch

### Node Details

#### Detect Intent
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript classifier for lightweight intent detection.
- **Configuration choices:**  
  The code:
  - reads the incoming message from `chatInput` or `input`,
  - trims whitespace,
  - checks for summary keywords using a regex,
  - checks help commands,
  - defaults to `expense` otherwise.
- **Key expressions or variables used:**  
  - `const chatInput = ($input.first().json.chatInput || $input.first().json.input || '').trim();`
  - Summary regex: `/\b(summary|report|total|spending|how much|breakdown|stats)\b/i`
  - Help regex: `/^(\?|help|\/help)$/i`
  - Output fields:
    - `chat_input`
    - `intent`
- **Input and output connections:**  
  - Input: `When chat message received`
  - Output: `Intent Switch`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - A normal sentence containing words like “total” or “report” may be misclassified as `summary`.
  - Messages such as “help me spent 500” will not be treated as help unless they exactly match the help regex.
  - Empty input becomes `expense` because it is neither summary nor help.
- **Sub-workflow reference:**  
  None

#### Intent Switch
- **Type and technical role:** `n8n-nodes-base.switch`  
  Branch router based on the `intent` field.
- **Configuration choices:**  
  Three string-equality rules:
  - `expense`
  - `summary`
  - `help`
- **Key expressions or variables used:**  
  - `={{ $json.intent }}`
- **Input and output connections:**  
  - Input: `Detect Intent`
  - Outputs:
    - Output 0 → `Read All Expenses`
    - Output 1 → `Read for Summary`
    - Output 2 → `Send Help`
- **Version-specific requirements:**  
  `typeVersion: 3.2`
- **Edge cases or potential failure types:**  
  - If `intent` is missing or unexpected, no branch will match.
  - Loose type validation is enabled, but this is low-risk because comparisons are string-based.
- **Sub-workflow reference:**  
  None

---

## 2.3 Block: Expense Intake Context Preparation

### Overview
This block loads existing rows from Google Sheets and prepares a single guaranteed context item for the AI parsing stage. It prevents empty-sheet cases from stopping the workflow.

### Nodes Involved
- Read All Expenses
- Prepare Data

### Node Details

#### Read All Expenses
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads all current expense rows from the target spreadsheet.
- **Configuration choices:**  
  - Spreadsheet: `Expense track`
  - Sheet: `Sheet1` (`gid=0`)
  - `alwaysOutputData: true` so the flow continues even if there are no rows.
- **Key expressions or variables used:**  
  No expressions in the node configuration itself.
- **Input and output connections:**  
  - Input: `Intent Switch` expense branch
  - Output: `Prepare Data`
- **Version-specific requirements:**  
  `typeVersion: 4.5`
- **Edge cases or potential failure types:**  
  - Google authentication failure
  - Spreadsheet or sheet not found
  - Permission denied
  - API quota issues
  - Column naming inconsistencies affecting later code
- **Sub-workflow reference:**  
  None

#### Prepare Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Prepares stable context for AI parsing and calculates the current month total from existing rows.
- **Configuration choices:**  
  The code:
  - reads all rows with `$input.all()`,
  - retrieves the original `chat_input` from `Detect Intent`,
  - computes `currentMonth` using the server locale logic,
  - sums existing rows whose `Month` equals the current month string,
  - always returns exactly one item.
- **Key expressions or variables used:**  
  - `$('Detect Intent').first().json.chat_input`
  - `const allRows = $input.all();`
  - current month format:
    - `now.toLocaleString('en-US', {month:'long'}) + ' ' + now.getFullYear()`
  - output fields:
    - `chat_input`
    - `month_total`
    - `current_month`
    - `row_count`
- **Input and output connections:**  
  - Input: `Read All Expenses`
  - Output: `AI Parse Expense`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - If sheet rows contain non-numeric values in `Amount`, `parseFloat` may produce `NaN`, potentially affecting totals.
  - Month matching relies on exact text equality with the `Month` column.
  - Timezone and locale differences may affect month naming if the runtime changes.
- **Sub-workflow reference:**  
  None

---

## 2.4 Block: AI Expense Parsing

### Overview
This block sends the natural-language message to Claude Haiku and requests a strict JSON object containing amount, description, category, currency, date, and an `is_expense` flag.

### Nodes Involved
- AI Parse Expense
- Claude Haiku

### Node Details

#### AI Parse Expense
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  LLM chain node that formulates a prompt and invokes the connected language model.
- **Configuration choices:**  
  - Uses `promptType: define`
  - Sends a strongly constrained parsing prompt
  - Requires output to be JSON only
  - Provides examples for valid and invalid expense messages
  - Defaults:
    - currency: `INR`
    - date: today
- **Key expressions or variables used:**  
  - Message source:
    - `{{ $('Detect Intent').first().json.chat_input }}`
  - Current date:
    - `{{ new Date().toISOString().split('T')[0] }}`
  - Expected JSON schema:
    - `amount`
    - `description`
    - `category`
    - `currency`
    - `date`
    - `is_expense`
- **Input and output connections:**  
  - Main input: `Prepare Data`
  - AI language model input: `Claude Haiku`
  - Output: `Parse & Total`
- **Version-specific requirements:**  
  `typeVersion: 1.4`
- **Edge cases or potential failure types:**  
  - The model may still return extra text despite “JSON only” instructions.
  - Ambiguous messages may be misclassified.
  - Date extraction may be inconsistent if the user describes past expenses informally.
  - Category may fall outside the allowed enum if model behavior drifts.
- **Sub-workflow reference:**  
  None

#### Claude Haiku
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Anthropic chat model provider node.
- **Configuration choices:**  
  - Model selected: `claude-haiku-4-5-20251001`
  - Connected as the language model for `AI Parse Expense`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI model output connection to `AI Parse Expense`
- **Version-specific requirements:**  
  `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Invalid or missing Anthropic credentials
  - Model access restrictions
  - Rate limits
  - Timeout or upstream API failures
  - Model identifier availability may vary by n8n version or Anthropic account permissions
- **Sub-workflow reference:**  
  None

---

## 2.5 Block: Expense Validation, Totaling, and Persistence

### Overview
This block parses the AI result, validates whether it contains a usable expense, computes the correct monthly running total based on the parsed date, and either saves the row to Google Sheets or returns an invalid-input response.

### Nodes Involved
- Parse & Total
- Is Valid Expense?
- Save Expense to Sheet
- Reply Saved
- Reply Invalid

### Node Details

#### Parse & Total
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts the AI response into structured sheet-ready data and generates the user-facing display text.
- **Configuration choices:**  
  The code:
  - reads `text` or `output` from the AI node,
  - attempts direct JSON parse,
  - falls back to extracting the first `{...}` block,
  - validates `is_expense` and `amount > 0`,
  - computes `Month` from the parsed date, not from today,
  - sums prior rows for that exact month,
  - builds a formatted `_display` string.
- **Key expressions or variables used:**  
  - `($json.text || $json.output || '').trim()`
  - `$('Prepare Data').first().json`
  - `$('Read All Expenses').all()`
  - generated output fields:
    - `valid`
    - `Date`
    - `Amount`
    - `Category`
    - `Description`
    - `Currency`
    - `Month`
    - `Raw Message`
    - `Total`
    - `_display`
    - or `reply` on invalid input
- **Input and output connections:**  
  - Input: `AI Parse Expense`
  - Output: `Is Valid Expense?`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - AI response may not parse as JSON.
  - `new Date(dateStr)` may produce invalid date behavior if the model emits malformed dates.
  - Existing sheet data with malformed `Amount` values may break numeric sums.
  - Currency symbol formatting only maps `INR` to `₹`; others use `CODE `.
  - If category is outside the emoji map, default emoji `💰` is used.
- **Sub-workflow reference:**  
  None

#### Is Valid Expense?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches on whether the parsed expense is valid.
- **Configuration choices:**  
  Boolean condition:
  - `{{ $json.valid }} == true`
- **Key expressions or variables used:**  
  - `={{ $json.valid }}`
- **Input and output connections:**  
  - Input: `Parse & Total`
  - True output: `Save Expense to Sheet`
  - False output: `Reply Invalid`
- **Version-specific requirements:**  
  `typeVersion: 2.1`
- **Edge cases or potential failure types:**  
  - If `valid` is absent or non-boolean, loose validation may still evaluate unexpectedly.
- **Sub-workflow reference:**  
  None

#### Save Expense to Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to the spreadsheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet: `Expense track`
  - Sheet: `Sheet1`
  - Explicit column mapping:
    - `Date`
    - `Month`
    - `Total`
    - `Amount`
    - `Category`
    - `Currency`
    - `Description`
    - `Raw Message`
  - `mappingMode: defineBelow`
- **Key expressions or variables used:**  
  - `={{ $json.Date }}`
  - `={{ $json.Month }}`
  - `={{ $json.Total }}`
  - `={{ $json.Amount }}`
  - `={{ $json.Category }}`
  - `={{ $json.Currency }}`
  - `={{ $json.Description }}`
  - `={{ $json['Raw Message'] }}`
- **Input and output connections:**  
  - Input: `Is Valid Expense?` true branch
  - Output: `Reply Saved`
- **Version-specific requirements:**  
  `typeVersion: 4.5`
- **Edge cases or potential failure types:**  
  - Column name mismatch in Google Sheets
  - Sheet permission issues
  - Append failures due to protected ranges
  - Type coercion surprises if sheet formatting is strict
- **Sub-workflow reference:**  
  None

#### Reply Saved
- **Type and technical role:** `n8n-nodes-base.code`  
  Builds the final confirmation message after a successful write.
- **Configuration choices:**  
  Reads the structured result from `Parse & Total` and formats a confirmation with `_display`.
- **Key expressions or variables used:**  
  - `$('Parse & Total').first().json`
  - Output field: `output`
- **Input and output connections:**  
  - Input: `Save Expense to Sheet`
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - If upstream data shape changes and `_display` is missing, the message becomes incomplete.
- **Sub-workflow reference:**  
  None

#### Reply Invalid
- **Type and technical role:** `n8n-nodes-base.code`  
  Returns a clarification message when the AI parse did not identify a valid expense amount.
- **Configuration choices:**  
  Reads the fallback `reply` field from `Parse & Total`, otherwise uses a default message.
- **Key expressions or variables used:**  
  - `$('Parse & Total').first().json`
  - `d.reply || "Please include an amount in your message."`
- **Input and output connections:**  
  - Input: `Is Valid Expense?` false branch
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - If `Parse & Total` output changes, `reply` may be missing.
- **Sub-workflow reference:**  
  None

---

## 2.6 Block: Summary Generation

### Overview
This block handles summary/report requests. It loads all stored expenses, detects the month requested in the chat message, aggregates totals and categories, and returns a formatted monthly report.

### Nodes Involved
- Read for Summary
- Build Summary

### Node Details

#### Read for Summary
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads all stored expense rows for reporting.
- **Configuration choices:**  
  - Spreadsheet: `Expense track`
  - Sheet: `Sheet1`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**  
  None in the node config.
- **Input and output connections:**  
  - Input: `Intent Switch` summary branch
  - Output: `Build Summary`
- **Version-specific requirements:**  
  `typeVersion: 4.5`
- **Edge cases or potential failure types:**  
  Same as other Google Sheets read operations:
  - auth failure
  - sheet not found
  - permission denied
  - empty result set
- **Sub-workflow reference:**  
  None

#### Build Summary
- **Type and technical role:** `n8n-nodes-base.code`  
  Determines the requested month and produces a human-readable monthly report.
- **Configuration choices:**  
  The code:
  - loads all rows,
  - reads `chat_input` from `Detect Intent`,
  - scans for English month names,
  - optionally detects a 4-digit year (`20xx`) or a 2-digit year,
  - defaults to the current month,
  - filters rows by exact `Month`,
  - sums totals and groups by category,
  - calculates entry count, daily average, top category, and percentage breakdown.
- **Key expressions or variables used:**  
  - `$('Detect Intent').first().json.chat_input`
  - `const rows = $input.all();`
  - month matching against row field `Month`
  - Output field: `output`
- **Input and output connections:**  
  - Input: `Read for Summary`
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Daily average uses `now.getDate()` even for past months, so the average is based on the current day number rather than the number of days in the target month or days with entries.
  - Summary formatting always uses `₹`, regardless of stored currency.
  - If the sheet contains multiple currencies, totals are mathematically combined without conversion.
  - Two-digit year parsing may interpret any standalone two-digit number as a year.
  - Month detection works only with English month names.
  - If no rows match the target month, the node returns a “No expenses found” message.
- **Sub-workflow reference:**  
  None

---

## 2.7 Block: Help Response

### Overview
This block returns usage guidance when the user sends `help`, `?`, or `/help`. It explains expense entry examples, reporting commands, supported currencies, and categories.

### Nodes Involved
- Send Help

### Node Details

#### Send Help
- **Type and technical role:** `n8n-nodes-base.code`  
  Produces a static help response.
- **Configuration choices:**  
  Returns one item containing a formatted `output` message.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Intent Switch` help branch
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  Very low risk; mostly limited to formatting changes.
- **Sub-workflow reference:**  
  None

---

## 2.8 Block: Documentation and Visual Notes

### Overview
These sticky notes document the workflow visually inside the n8n canvas. They do not affect execution but provide important operational context, including commands and expected sheet columns.

### Nodes Involved
- Overview
- Trigger Note
- Intent Note
- Switch Note
- Read Note
- Prepare Note
- AI Note
- Parse Note
- Valid Note
- Save Note
- Summary Note
- Sheet Columns

### Node Details

#### Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note.
- **Configuration choices:**  
  Describes the workflow purpose, commands, and supported currencies.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Trigger Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the entry point and `chatInput` handoff.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Intent Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents intent classification.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Switch Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents switch output routing.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Read Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the sheet read behavior and `alwaysOutputData`.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Prepare Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents one-item output behavior.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### AI Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the AI extraction fields and JSON-only expectation.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Parse Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents validation and month derivation.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Valid Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the true/false branch behavior.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Save Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents append and confirmation behavior.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Summary Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the summary flow and month detection.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sheet Columns
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the intended spreadsheet column layout.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry trigger |  | Detect Intent | ## 1️⃣ Entry Point<br>Receives every chat message.<br>Passes `chatInput` to next node. |
| Detect Intent | n8n-nodes-base.code | Classify message as expense, summary, or help | When chat message received | Intent Switch | ## 2️⃣ Detect Intent<br>Reads the message and classifies:<br>• **expense** — anything with an amount<br>• **summary** — report/total/breakdown<br>• **help** — help/? |
| Intent Switch | n8n-nodes-base.switch | Route execution to expense, summary, or help branch | Detect Intent | Read All Expenses; Read for Summary; Send Help | ## 3️⃣ Routes to 3 paths<br>**Output 0** → Expense flow<br>**Output 1** → Summary flow<br>**Output 2** → Help message |
| Read All Expenses | n8n-nodes-base.googleSheets | Load all existing expense rows before parsing new expense | Intent Switch | Prepare Data | ## 4️⃣ Read Sheet<br>Loads all existing rows.<br>`alwaysOutputData: true` ensures<br>flow continues even if sheet is empty.<br><br>## 📋 Sheet Columns<br>A: Date<br>B: Amount<br>C: Category<br>D: Description<br>E: Currency<br>F: Month<br>G: Raw Message<br>H: Total _(running monthly)_ |
| Prepare Data | n8n-nodes-base.code | Build one-item context and current-month total | Read All Expenses | AI Parse Expense | ## 5️⃣ Prepare Data<br>Computes running month total<br>from existing rows.<br>Always outputs 1 item → flow<br>never stops on empty sheet. |
| AI Parse Expense | @n8n/n8n-nodes-langchain.chainLlm | Prompt Claude to extract structured expense JSON | Prepare Data; Claude Haiku (AI model) | Parse & Total | ## 6️⃣ AI Parse<br>Claude Haiku reads the message<br>and extracts:<br>• amount · description<br>• category · currency · date<br>• is_expense (true/false)<br><br>Returns clean JSON only. |
| Claude Haiku | @n8n/n8n-nodes-langchain.lmChatAnthropic | Anthropic model backend for parsing |  | AI Parse Expense | ## 6️⃣ AI Parse<br>Claude Haiku reads the message<br>and extracts:<br>• amount · description<br>• category · currency · date<br>• is_expense (true/false)<br><br>Returns clean JSON only. |
| Parse & Total | n8n-nodes-base.code | Parse AI output, validate expense, compute month total, prepare display | AI Parse Expense | Is Valid Expense? | ## 7️⃣ Parse & Total<br>Validates AI response.<br>Derives Month from parsed date<br>(not today — handles past entries).<br>Calculates running monthly total. |
| Is Valid Expense? | n8n-nodes-base.if | Branch valid vs invalid expense messages | Parse & Total | Save Expense to Sheet; Reply Invalid | ## 8️⃣ Valid?<br>✅ TRUE → Save to sheet<br>❌ FALSE → Ask user<br>to add an amount |
| Save Expense to Sheet | n8n-nodes-base.googleSheets | Append validated expense row to Google Sheets | Is Valid Expense? | Reply Saved | ## 9️⃣ Save + Reply<br>Appends new row to sheet.<br>Replies with confirmation:<br>✅ Category · Amount · Date<br>📊 Running month total<br><br>## 📋 Sheet Columns<br>A: Date<br>B: Amount<br>C: Category<br>D: Description<br>E: Currency<br>F: Month<br>G: Raw Message<br>H: Total _(running monthly)_ |
| Reply Saved | n8n-nodes-base.code | Return confirmation after successful sheet append | Save Expense to Sheet |  | ## 9️⃣ Save + Reply<br>Appends new row to sheet.<br>Replies with confirmation:<br>✅ Category · Amount · Date<br>📊 Running month total |
| Reply Invalid | n8n-nodes-base.code | Return clarification when no expense amount was identified | Is Valid Expense? |  | ## 9️⃣ Save + Reply<br>Appends new row to sheet.<br>Replies with confirmation:<br>✅ Category · Amount · Date<br>📊 Running month total |
| Read for Summary | n8n-nodes-base.googleSheets | Load all expenses for reporting | Intent Switch | Build Summary | ## 📊 SUMMARY FLOW<br>Reads all rows → filters by month.<br>Detects specific month from message<br>_(e.g. "summary february 2026")_<br>Defaults to current month.<br>Shows total, breakdown, daily avg. |
| Build Summary | n8n-nodes-base.code | Build monthly report from sheet data | Read for Summary |  | ## 📊 SUMMARY FLOW<br>Reads all rows → filters by month.<br>Detects specific month from message<br>_(e.g. "summary february 2026")_<br>Defaults to current month.<br>Shows total, breakdown, daily avg. |
| Send Help | n8n-nodes-base.code | Return supported commands and input examples | Intent Switch |  |  |
| Overview | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Trigger Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Intent Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Switch Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Read Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Prepare Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| AI Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Parse Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Valid Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Save Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Summary Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sheet Columns | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Chat Trigger node**
   - Node type: `When chat message received`
   - Actual node: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Leave default options unless your chat surface requires custom settings.
   - This will be the workflow entry point.

3. **Add a Code node named `Detect Intent`**
   - Node type: `n8n-nodes-base.code`
   - Connect: `When chat message received` → `Detect Intent`
   - Paste this logic conceptually:
     - Read the message from `chatInput` or fallback to `input`
     - Trim whitespace
     - If it contains summary-like keywords, set `intent = summary`
     - If it exactly matches `?`, `help`, or `/help`, set `intent = help`
     - Otherwise set `intent = expense`
   - Output fields must be:
     - `chat_input`
     - `intent`

4. **Add a Switch node named `Intent Switch`**
   - Node type: `n8n-nodes-base.switch`
   - Connect: `Detect Intent` → `Intent Switch`
   - Create 3 rules using `{{ $json.intent }}`:
     1. equals `expense`
     2. equals `summary`
     3. equals `help`

5. **Prepare Google Sheets storage**
   - Create a Google Sheet named something like `Expense track`.
   - Use one worksheet, for example `Sheet1`.
   - Create these columns in the header row:
     - `Date`
     - `Amount`
     - `Category`
     - `Description`
     - `Currency`
     - `Month`
     - `Raw Message`
     - `Total`
   - Make sure the column names match exactly.

6. **Configure Google Sheets credentials**
   - In n8n, create or select Google Sheets credentials with access to the target spreadsheet.
   - Ensure the n8n-connected Google account can read and append rows.

7. **Add a Google Sheets node named `Read All Expenses`**
   - Node type: `n8n-nodes-base.googleSheets`
   - Connect: `Intent Switch` expense output → `Read All Expenses`
   - Operation: read rows from the sheet
   - Select:
     - Document: your expense spreadsheet
     - Sheet: `Sheet1`
   - Enable `Always Output Data`.

8. **Add a Code node named `Prepare Data`**
   - Connect: `Read All Expenses` → `Prepare Data`
   - Implement logic to:
     - read all rows using `$input.all()`
     - get `chat_input` from `Detect Intent`
     - compute current month as `"MonthName YYYY"`
     - sum `Amount` values where `Month` equals the current month
     - always return exactly one item
   - Output fields:
     - `chat_input`
     - `month_total`
     - `current_month`
     - `row_count`

9. **Add an Anthropic Chat Model node named `Claude Haiku`**
   - Node type: `@n8n/n8n-nodes-langchain.lmChatAnthropic`
   - Configure Anthropic credentials.
   - Select a Claude Haiku model available in your account.
   - In the source workflow the selected model is `claude-haiku-4-5-20251001`.
   - If that exact model is not available, choose a compatible Claude Haiku variant.

10. **Add an LLM Chain node named `AI Parse Expense`**
    - Node type: `@n8n/n8n-nodes-langchain.chainLlm`
    - Connect main flow: `Prepare Data` → `AI Parse Expense`
    - Connect AI model port: `Claude Haiku` → `AI Parse Expense`
    - Use a defined prompt.
    - The prompt should:
      - identify itself as an expense parser,
      - include the user message from `{{ $('Detect Intent').first().json.chat_input }}`,
      - include today’s date,
      - require JSON-only output,
      - enforce these fields:
        - `amount`
        - `description`
        - `category`
        - `currency`
        - `date`
        - `is_expense`
      - include allowed categories:
        - Food & Dining
        - Transport
        - Shopping
        - Bills & Utilities
        - Entertainment
        - Health
        - Business
        - Education
        - Other
      - default currency to `INR`
      - default date to today
      - include positive and negative examples

11. **Add a Code node named `Parse & Total`**
    - Connect: `AI Parse Expense` → `Parse & Total`
    - Implement logic to:
      - read model output from `text` or `output`
      - parse JSON directly
      - if direct parse fails, extract the first JSON object from text and parse it
      - reject outputs where:
        - JSON is invalid
        - `is_expense` is false
        - `amount <= 0`
      - on rejection, output:
        - `valid: false`
        - a user-friendly `reply`
      - on success:
        - parse `amount`
        - determine `Date`
        - derive `Month` from parsed date, not today
        - sum all existing rows in the same month from `Read All Expenses`
        - compute `Total = existingMonthTotal + amount`
        - build fields:
          - `Date`
          - `Amount`
          - `Category`
          - `Description`
          - `Currency`
          - `Month`
          - `Raw Message`
          - `Total`
          - `_display`
        - set `valid: true`

12. **Add an IF node named `Is Valid Expense?`**
    - Node type: `n8n-nodes-base.if`
    - Connect: `Parse & Total` → `Is Valid Expense?`
    - Condition:
      - `{{ $json.valid }}` equals boolean `true`

13. **Add a Google Sheets node named `Save Expense to Sheet`**
    - Node type: `n8n-nodes-base.googleSheets`
    - Connect: `Is Valid Expense?` true branch → `Save Expense to Sheet`
    - Operation: `append`
    - Select the same spreadsheet and sheet.
    - Use explicit field mapping.
    - Map:
      - `Date` ← `{{ $json.Date }}`
      - `Month` ← `{{ $json.Month }}`
      - `Total` ← `{{ $json.Total }}`
      - `Amount` ← `{{ $json.Amount }}`
      - `Category` ← `{{ $json.Category }}`
      - `Currency` ← `{{ $json.Currency }}`
      - `Description` ← `{{ $json.Description }}`
      - `Raw Message` ← `{{ $json['Raw Message'] }}`

14. **Add a Code node named `Reply Saved`**
    - Connect: `Save Expense to Sheet` → `Reply Saved`
    - Build output:
      - read data from `Parse & Total`
      - return `{ output: "✅ Expense saved!\n\n" + _display }`

15. **Add a Code node named `Reply Invalid`**
    - Connect: `Is Valid Expense?` false branch → `Reply Invalid`
    - Return:
      - `reply` from `Parse & Total` if present
      - otherwise a default message asking the user to include an amount

16. **Add a second Google Sheets node named `Read for Summary`**
    - Node type: `n8n-nodes-base.googleSheets`
    - Connect: `Intent Switch` summary output → `Read for Summary`
    - Select the same spreadsheet and same sheet.
    - Enable `Always Output Data`.

17. **Add a Code node named `Build Summary`**
    - Connect: `Read for Summary` → `Build Summary`
    - Implement logic to:
      - read all rows
      - get `chat_input` from `Detect Intent`
      - detect an English month name inside the message
      - optionally detect year:
        - 4-digit year like `2026`
        - or 2-digit year like `26`
      - if no month is mentioned, default to current month
      - filter rows by exact match on `Month`
      - if no rows match, return a “No expenses found” message
      - otherwise compute:
        - monthly total
        - number of entries
        - spending by category
        - top category
        - category percentages
        - daily average
      - return a formatted `output` string

18. **Add a Code node named `Send Help`**
    - Connect: `Intent Switch` help output → `Send Help`
    - Return a static `output` text including:
      - sample expense phrases
      - summary command
      - supported currencies
      - category list

19. **Optional: add sticky notes for maintainability**
    - Add notes documenting:
      - entry point
      - intent routing
      - sheet column schema
      - AI parsing contract
      - validation logic
      - summary behavior

20. **Test the expense path**
    - Example inputs:
      - `spent 500 on lunch`
      - `uber 150`
      - `paid 1200 electricity bill`
    - Confirm:
      - model returns parsable JSON
      - row is appended
      - confirmation text appears

21. **Test invalid expense handling**
    - Example inputs:
      - `hello`
      - `track this`
    - Confirm the invalid branch asks the user to include an amount.

22. **Test summary path**
    - Example inputs:
      - `summary`
      - `summary february`
      - `report march 2026`
    - Confirm:
      - rows are filtered by `Month`
      - output includes total, entries, daily average, and category breakdown

23. **Test help path**
    - Example inputs:
      - `help`
      - `?`
      - `/help`

24. **Credential expectations**
    - Required credentials:
      - Google Sheets
      - Anthropic
    - Make sure both credentials are authorized before enabling the workflow.

25. **Important rebuild constraints**
    - Keep Google Sheet header names exactly consistent with the append mapping.
    - Keep `alwaysOutputData` enabled on both read nodes.
    - Ensure `Prepare Data` always returns one item.
    - Ensure the AI prompt explicitly asks for JSON only.
    - Keep month formatting consistent as `"MonthName YYYY"` across write and summary logic.

26. **No sub-workflow setup is required**
    - This workflow does not call any sub-workflow.
    - There are no Execute Workflow nodes or reusable child workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## 💰 AI Expense Tracker Track expenses by chatting naturally. Just type your expense: _"spent 500 on lunch"_ _"uber 150"_ _"1200 electricity bill"_ Commands: • `SUMMARY` — current month report • `summary february` — specific month • `HELP` — all commands • `Currencies:` ₹ INR · $ USD · € EUR · £ GBP | Canvas overview note |
| Google Sheet structure expected by the workflow: `Date`, `Amount`, `Category`, `Description`, `Currency`, `Month`, `Raw Message`, `Total` | Required for correct reads, summaries, and appends |
| Summary logic assumes English month names in user messages. | Important for month detection |
| Totals in summaries are not currency-normalized. Mixed currencies will be added together as raw numbers. | Design limitation |
| The summary output always displays `₹` in the report formatting, even if stored expenses use other currencies. | Formatting limitation |
| Daily average is computed using the current day of the month, not the full target month length or number of days with expenses. | Reporting caveat |
| The workflow contains one entry point only: the chat trigger. | Architecture note |