Query expenses on Telegram with GPT-4.1 and Google Sheets

https://n8nworkflows.xyz/workflows/query-expenses-on-telegram-with-gpt-4-1-and-google-sheets-14051


# Query expenses on Telegram with GPT-4.1 and Google Sheets

# 1. Workflow Overview

This workflow turns Telegram messages into expense queries against data stored in Google Sheets. A user can ask questions such as “How much did I spend on food last month?” or “Show shared expenses by category this week”, and the workflow will:

1. receive the Telegram event,
2. authorize the sender,
3. parse the natural-language query with GPT-4.1-nano,
4. resolve category and person names to canonical values,
5. load expense data from Google Sheets,
6. filter and aggregate the data,
7. send a formatted Telegram reply.

It also supports interactive correction loops for ambiguous categories and person names. If an entity is unknown, the workflow asks the user to confirm an AI suggestion or select from inline Telegram buttons, then stores that mapping for future use.

## 1.1 Input Reception, Authorization, and Callback Routing
This block receives both Telegram messages and callback queries, checks whether the sender is authorized, and routes either into the main query flow or back into a waiting branch for inline-button selection handling.

## 1.2 Intent Parsing
This block uses OpenAI GPT-4.1-nano to convert the free-text Telegram message into strict structured JSON representing the user’s query intent.

## 1.3 Category Resolution
This block resolves category aliases or unknown category values into canonical categories using:
- a Google Sheets alias mapping,
- a canonical allowed category list,
- an LLM fallback,
- Telegram user confirmation,
- and persistent learning via saved mappings.

## 1.4 Person Resolution
This block mirrors the category logic for person names. It resolves aliases, validates persons against an allowed list, uses AI when needed, and stores approved mappings.

## 1.5 Intent Reassembly and Data Loading
This block merges the resolved person/category data back into the parsed intent, then loads expenses and category reference data from Google Sheets.

## 1.6 Query Execution and Aggregation
This block computes the actual result: date-range resolution, filtering, totals, counts, and optional breakdowns by category or person.

## 1.7 Response Formatting and Delivery
This block formats a Telegram-friendly Markdown response and replies to the original Telegram message.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception, Authorization, and Callback Routing

### Overview
This block is the workflow entry layer. It listens for incoming Telegram messages and callback queries, restricts usage to approved chat IDs, and routes callback selections back to the correct paused wait node.

### Nodes Involved
- `MSG | Telegram Inbound`
- `IF | Message or Callback?`
- `IF | User Authorized?`
- `IF | Category or Person Callback?`
- `JS | Read Category Callback`
- `HTTP | Forward Category Selection`
- `TG | Confirm Category Selection`
- `JS | Read Person Callback`
- `HTTP | Forward Person Selection`
- `TG | Confirm Person Selection`

### Node Details

#### MSG | Telegram Inbound
- **Type / Role:** `telegramTrigger`; workflow entry point for Telegram events.
- **Configuration:** Listens for both `message` and `callback_query` updates.
- **Key expressions / variables:** Uses Telegram credentials.
- **Input / Output:** No input; outputs to `IF | Message or Callback?`.
- **Version-specific requirements:** Type version 1.1.
- **Failure / edge cases:**
  - Telegram credential missing or invalid.
  - Telegram webhook registration issues.
  - Callback payloads may differ from message payloads, so downstream expressions must branch carefully.

#### IF | Message or Callback?
- **Type / Role:** `if`; distinguishes standard message events from callback queries.
- **Configuration:** Checks whether `$json.message` exists.
- **Key expressions / variables:** `={{ $json.message }}`
- **Input / Output:** Input from Telegram trigger.
  - True branch: `IF | User Authorized?`
  - False branch: `IF | Category or Person Callback?`
- **Failure / edge cases:**
  - Telegram callback events do not contain `message` in the same way as normal messages.
  - If Telegram payload shape changes, this routing may misclassify events.

#### IF | User Authorized?
- **Type / Role:** `if`; access control.
- **Configuration:** Compares `message.chat.id` against two placeholder numeric IDs using OR logic.
- **Key expressions / variables:** `={{ $json.message.chat.id }}`
- **Input / Output:** Input from `IF | Message or Callback?`; authorized branch goes to `LLM | Parse Intent`.
- **Version-specific requirements:** Version 2.2 conditions syntax.
- **Failure / edge cases:**
  - Placeholder IDs must be replaced.
  - Unauthorized users are silently ignored because no false branch is connected.
  - Group chats may have different chat IDs than expected.

#### IF | Category or Person Callback?
- **Type / Role:** `if`; routes callback replies to category or person selection handlers.
- **Configuration:** Checks whether the callback message text contains `Kategorie`.
- **Key expressions / variables:** `{{json.message.text}}`
- **Input / Output:** Input from `IF | Message or Callback?`.
  - True branch: `JS | Read Category Callback`
  - False branch: `JS | Read Person Callback`
- **Failure / edge cases:**
  - Uses message text language-dependent detection.
  - If the inline prompt text changes or is translated, routing may fail.
  - The expression appears to rely on callback message text structure.

#### JS | Read Category Callback
- **Type / Role:** `code`; extracts category selection and resume URL from callback.
- **Configuration:** Reads:
  - selected category from `callback_query.data`
  - resume URL from a `text_link` message entity.
- **Key expressions / variables:** `callback_query.data`, `callback_query.message.entities`
- **Input / Output:** Input from `IF | Category or Person Callback?`; output to `HTTP | Forward Category Selection`.
- **Failure / edge cases:**
  - Fails if no `text_link` entity exists.
  - Assumes the original callback message included an embedded link.
  - `entities.find(...)` can throw if entities are missing.

#### HTTP | Forward Category Selection
- **Type / Role:** `httpRequest`; resumes a waiting execution branch.
- **Configuration:** POSTs `{ "category": "<selected>" }` to the extracted resume URL.
- **Key expressions / variables:** `={{ $json.resumeUrl }}`
- **Input / Output:** Input from `JS | Read Category Callback`; output to `TG | Confirm Category Selection`.
- **Failure / edge cases:**
  - Resume URL expired or invalid.
  - Internal n8n wait resume endpoint inaccessible.
  - Network issues or non-200 responses.

#### TG | Confirm Category Selection
- **Type / Role:** `telegram`; sends acknowledgment after callback processing.
- **Configuration:** Sends a plain confirmation message to the callback chat.
- **Key expressions / variables:**
  - Chat ID from callback message chat
  - Selected value from `callback_query.data`
- **Input / Output:** Input from `HTTP | Forward Category Selection`; terminal in this branch.
- **Failure / edge cases:**
  - Telegram credential/auth issues.
  - Message text contains a minor formatting anomaly with quotes.

#### JS | Read Person Callback
- **Type / Role:** `code`; extracts person selection and resume URL from callback.
- **Configuration:** Same pattern as category callback extractor.
- **Key expressions / variables:** `callback_query.data`, `callback_query.message.entities`
- **Input / Output:** Input from `IF | Category or Person Callback?`; output to `HTTP | Forward Person Selection`.
- **Failure / edge cases:**
  - Same risks as category callback extraction.
  - Missing `entities` or missing `text_link` breaks the node.

#### HTTP | Forward Person Selection
- **Type / Role:** `httpRequest`; resumes paused person selection branch.
- **Configuration:** POSTs `{ "person": "<selected>" }` to resume URL.
- **Input / Output:** Input from `JS | Read Person Callback`; output to `TG | Confirm Person Selection`.
- **Failure / edge cases:**
  - Invalid resume URL or wait timeout.
  - Internal webhook/network access failure.

#### TG | Confirm Person Selection
- **Type / Role:** `telegram`; sends callback acknowledgment.
- **Configuration:** Replies with selected person text.
- **Input / Output:** Input from `HTTP | Forward Person Selection`; terminal in this branch.
- **Failure / edge cases:** Same as other Telegram send nodes.

---

## 2.2 Intent Parsing

### Overview
This block converts the incoming Telegram message into a strict JSON query object. The LLM is heavily constrained by the system prompt, and a code node sanitizes and parses the response.

### Nodes Involved
- `LLM | Parse Intent`
- `JS | Extract Intent JSON`

### Node Details

#### LLM | Parse Intent
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi`; structured intent extraction with GPT-4.1-nano.
- **Configuration:**
  - Model: `gpt-4.1-nano`
  - System prompt instructs the model to return only JSON.
  - User message is the Telegram message text.
  - Prompt includes:
    - current user first name,
    - current year,
    - current date,
    - strict schema for intent fields.
- **Key expressions / variables:**
  - `{{ $('MSG | Telegram Inbound').item.json.message.from.first_name }}`
  - `{{ new Date().getFullYear() }}`
  - `{{ new Date().toISOString().split('T')[0] }}`
  - `={{ $json.message.text }}`
- **Input / Output:** Input from authorized branch; output to `JS | Extract Intent JSON`.
- **Version-specific requirements:** Type version 1.4.
- **Failure / edge cases:**
  - OpenAI credential missing or invalid.
  - Model availability or quota issues.
  - Non-JSON output despite prompt constraints.
  - Prompt assumes inbound event is a normal message, not callback.

#### JS | Extract Intent JSON
- **Type / Role:** `code`; post-processes LLM output.
- **Configuration:**
  - Reads response content from either:
    - `json.message.content`
    - or `json.choices[0].message.content`
  - Removes Markdown code fences.
  - Attempts JSON parse.
  - Falls back to a default intent if parsing fails.
- **Key expressions / variables:** Uses `$input.first()`.
- **Input / Output:** Input from `LLM | Parse Intent`; outputs to:
  - `IF | Category Present?`
  - `IF | Person Present?`
- **Failure / edge cases:**
  - Fallback object uses `start_date` / `end_date`, while downstream logic expects `explicit_start_date` / `explicit_end_date`; this inconsistency may reduce correctness on parse failure.
  - Invalid JSON or empty content is silently tolerated.

---

## 2.3 Category Resolution

### Overview
This block resolves requested categories to valid canonical categories from the reference sheet. It first tries alias mapping, then validation against the allowed list, then AI suggestion and human confirmation if necessary.

### Nodes Involved
- `IF | Category Present?`
- `SPLIT | Split Categories`
- `GS | Read Category Mapping`
- `MERGE | Join Categories with Mapping`
- `SET | Normalize Category`
- `GS | Read Allowed Categories`
- `MERGE | Check Category against Allowed`
- `IF | Category Known?`
- `AGG | Aggregate Category List`
- `MERGE | Combine Categories with List`
- `LOOP | Iterate Categories`
- `LLM | Classify Category`
- `SET | Extract LLM Category`
- `TG | Confirm Category Suggestion`
- `IF | Category Suggestion Accepted?`
- `JS | Build Category Inline Buttons`
- `HTTP | Send Category Selection Message`
- `WAIT | Wait for Category Selection`
- `SET | Read Callback Body (Cat.)`
- `SET | Set New+Old Category (Selection)`
- `SET | Set New+Old Category (LLM)`
- `MERGE | Combine Category Mapping Entries`
- `GS | Save Category Mapping`
- `MERGE | Loop Entry (Category)`
- `SET | Set Resolved Category`
- `AGG | Aggregate Resolved Categories`

### Node Details

#### IF | Category Present?
- **Type / Role:** `if`; determines whether category resolution is needed.
- **Configuration:** Tests whether `$json.category[0]` does not exist.
- **Behavior:** The true branch skips resolution and goes directly toward intent assembly; the false branch starts category processing.
- **Potential issue:** The condition logic is inverted relative to the label. It appears to treat missing category as the first branch.
- **Input / Output:**
  - Branch 0: `MERGE | Combine Category + Person`
  - Branch 1: category resolution nodes
- **Failure / edge cases:** If `category` is a string instead of an array, this check may behave unexpectedly.

#### SPLIT | Split Categories
- **Type / Role:** `splitOut`; processes each category independently.
- **Configuration:** Splits field `category`.
- **Input / Output:** Output to `MERGE | Join Categories with Mapping`.
- **Failure / edge cases:** Non-array category values may produce unexpected items.

#### GS | Read Category Mapping
- **Type / Role:** `googleSheets`; loads alias-to-canonical category mapping.
- **Configuration:** Reads from spreadsheet `categories_mapping`.
- **Expected columns:** `find`, `replace`
- **Input / Output:** To `MERGE | Join Categories with Mapping`.
- **Version:** 4.7
- **Failure / edge cases:**
  - Spreadsheet ID not replaced.
  - Missing columns.
  - OAuth2 credential failure.

#### MERGE | Join Categories with Mapping
- **Type / Role:** `merge` with SQL; joins raw category item to mapping table.
- **Configuration:** `SELECT * FROM input1 LEFT JOIN input2 ON input1.category = input2.find`
- **Input / Output:** Inputs from split categories and mapping sheet; output to `SET | Normalize Category`.
- **Failure / edge cases:**
  - Case-sensitive join may miss aliases.
  - Whitespace mismatches are not normalized before join.

#### SET | Normalize Category
- **Type / Role:** `set`; selects mapped category if available, otherwise original.
- **Configuration:** Sets `category_new = replace ?? category`
- **Input / Output:** Output to `MERGE | Check Category against Allowed`.
- **Failure / edge cases:** No lowercasing here, so matching depends on exact sheet content.

#### GS | Read Allowed Categories
- **Type / Role:** `googleSheets`; loads canonical category list.
- **Configuration:** Reads from `expense_categories`.
- **Expected columns:** `category`, `description`, `examples`
- **Outputs:** To `MERGE | Check Category against Allowed` and `AGG | Aggregate Category List`.
- **Failure / edge cases:** Missing or malformed category sheet.

#### MERGE | Check Category against Allowed
- **Type / Role:** `merge` with SQL; validates normalized category against allowed master list.
- **Configuration:** Left join on `category_new = category`
- **Input / Output:** Output to `IF | Category Known?`
- **Failure / edge cases:** Exact-string matching only.

#### IF | Category Known?
- **Type / Role:** `if`; checks whether category matched the allowed list.
- **Configuration:** Tests whether `description` does not exist.
- **Behavior:** A matched allowed category should include `description`; absence means unknown category.
- **Outputs:**
  - Branch 0: `MERGE | Combine Categories with List`
  - Branch 1: `MERGE | Loop Entry (Category)`
- **Potential issue:** Branch naming and logical semantics are easy to misread; actual behavior depends on how joins populate fields.

#### AGG | Aggregate Category List
- **Type / Role:** `aggregate`; creates one array of all allowed categories.
- **Configuration:** Aggregates field `category`.
- **Output:** `MERGE | Combine Categories with List`

#### MERGE | Combine Categories with List
- **Type / Role:** `merge`; combines unresolved item with allowed category list for LLM classification.
- **Configuration:** `combineAll`
- **Output:** `LOOP | Iterate Categories`

#### LOOP | Iterate Categories
- **Type / Role:** `splitInBatches`; iterates unresolved categories one by one.
- **Outputs:**
  - Loop continuation
  - Current item to `LLM | Classify Category`
- **Failure / edge cases:** If no items, loop behavior depends on upstream data flow.

#### LLM | Classify Category
- **Type / Role:** OpenAI node; maps unknown category to one allowed category.
- **Configuration:**
  - Model: `gpt-4.1-nano`
  - Strict prompt: return exactly one category from allowed list.
- **Key expressions / variables:**
  - Input category from `IF | Category Known?`
  - Allowed list from `$json.category`
- **Output:** `SET | Extract LLM Category`
- **Failure / edge cases:**
  - OpenAI may still return non-exact strings.
  - Long category list can reduce reliability.

#### SET | Extract LLM Category
- **Type / Role:** `set` raw mode; reshapes the LLM result.
- **Configuration:** `jsonOutput = {{ $json.message }}`
- **Output:** `TG | Confirm Category Suggestion`
- **Failure / edge cases:** Depends on OpenAI response shape; if `message` is not a JSON object, structure may not match expectations.

#### TG | Confirm Category Suggestion
- **Type / Role:** `telegram`; sends a Yes/No approval request.
- **Configuration:** Uses `sendAndWait` with double approval buttons.
- **Prompt:** “I couldn’t find the category X. Did you mean Y?”
- **Output:** `IF | Category Suggestion Accepted?`
- **Failure / edge cases:**
  - Waits for Telegram response; execution can remain paused.
  - Credential and webhook issues can interrupt the approval flow.

#### IF | Category Suggestion Accepted?
- **Type / Role:** `if`; checks approval result.
- **Configuration:** `{{$json.data.approved}} === true`
- **Outputs:**
  - True: `SET | Set New+Old Category (LLM)`
  - False: `JS | Build Category Inline Buttons`

#### JS | Build Category Inline Buttons
- **Type / Role:** `code`; builds Telegram inline keyboard from allowed categories.
- **Configuration:** Creates one button row per category and attaches `$execution.resumeUrl`.
- **Output:** `HTTP | Send Category Selection Message`
- **Failure / edge cases:** Throws if aggregated category list is not an array.

#### HTTP | Send Category Selection Message
- **Type / Role:** `httpRequest`; manually calls Telegram Bot API.
- **Configuration:**
  - Sends inline keyboard to user.
  - Embeds `resumeUrl` as hidden HTML link in message body.
  - Requires direct bot token in URL.
- **Output:** `WAIT | Wait for Category Selection`
- **Failure / edge cases:**
  - `{{YOUR_BOT_TOKEN}}` must be replaced manually.
  - HTML formatting or Telegram API errors.
  - Hidden link method is clever but brittle if Telegram alters entity handling.

#### WAIT | Wait for Category Selection
- **Type / Role:** `wait`; pauses execution until resumed via webhook.
- **Configuration:** Resume mode `webhook`, HTTP method POST.
- **Output:** `SET | Read Callback Body (Cat.)`
- **Failure / edge cases:**
  - Resume URL expires.
  - n8n instance must be publicly reachable if needed in your deployment mode.

#### SET | Read Callback Body (Cat.)
- **Type / Role:** `set` raw mode; unwraps the resume request body.
- **Configuration:** `jsonOutput = {{ $json.body }}`
- **Output:** `SET | Set New+Old Category (Selection)`

#### SET | Set New+Old Category (Selection)
- **Type / Role:** `set`; prepares old/new mapping from manual selection.
- **Configuration:**
  - `new = selected category`
  - `old = original unknown category`
- **Output:** `MERGE | Combine Category Mapping Entries`

#### SET | Set New+Old Category (LLM)
- **Type / Role:** `set`; prepares old/new mapping from approved AI suggestion.
- **Configuration:**
  - `new = extracted LLM content`
  - `old = original unknown category`
- **Output:** `MERGE | Combine Category Mapping Entries`

#### MERGE | Combine Category Mapping Entries
- **Type / Role:** `merge`; converges LLM-confirmed and manual-selection mapping paths.
- **Output:** `GS | Save Category Mapping`

#### GS | Save Category Mapping
- **Type / Role:** `googleSheets`; appends learned alias mapping.
- **Configuration:** Appends `find = old`, `replace = new` to `categories_mapping`.
- **Output:** loops back to `LOOP | Iterate Categories`
- **Failure / edge cases:**
  - Duplicate mappings may accumulate.
  - Spreadsheet write permissions required.

#### MERGE | Loop Entry (Category)
- **Type / Role:** `merge`; loop re-entry point for known or newly mapped category.
- **Output:** `SET | Set Resolved Category`

#### SET | Set Resolved Category
- **Type / Role:** `set`; emits final canonical category.
- **Configuration:** `category_new = replace ?? original category_new`
- **Output:** `AGG | Aggregate Resolved Categories`

#### AGG | Aggregate Resolved Categories
- **Type / Role:** `aggregate`; rebuilds final category list after processing all items.
- **Configuration:** Aggregates `category_new`
- **Output:** `MERGE | Combine Category + Person`

---

## 2.4 Person Resolution

### Overview
This block applies the same canonicalization strategy to person names. It supports aliases, unknown-name suggestions, approval, button-based selection, and mapping persistence.

### Nodes Involved
- `IF | Person Present?`
- `SPLIT | Split Persons`
- `GS | Read Person Mapping`
- `MERGE | Join Persons with Mapping`
- `SET | Normalize Person`
- `GS | Read Allowed Persons`
- `MERGE | Check Person against Allowed`
- `IF | Person Known?`
- `AGG | Aggregate Person List`
- `MERGE | Combine Persons with List`
- `LOOP | Iterate Persons`
- `LLM | Classify Person`
- `SET | Extract LLM Person`
- `TG | Confirm Person Suggestion`
- `IF | Person Suggestion Accepted?`
- `JS | Build Person Inline Buttons`
- `HTTP | Send Person Selection Message`
- `WAIT | Wait for Person Selection`
- `SET | Read Callback Body (Person)`
- `SET | Set New+Old Person (Selection)`
- `SET | Set New+Old Person (LLM)`
- `MERGE | Combine Person Mapping Entries`
- `GS | Save Person Mapping`
- `MERGE | Loop Entry (Person)`
- `SET | Set Resolved Person`
- `MERGE | Combine Person Mapping`
- `eaee6930-3da6-40a9-ba0f-84ab8543b451` node name shown as `MERGE | Loop Entry (Person)` already included

### Node Details

#### IF | Person Present?
- **Type / Role:** `if`; determines whether person resolution is needed.
- **Configuration:** Tests whether `$json.person` does not exist.
- **Behavior:** One branch skips person resolution; the other launches it.
- **Output:** Skip branch to `MERGE | Combine Person Mapping`; processing branch to split + sheet reads.
- **Potential issue:** Similar inversion ambiguity as the category presence node.

#### SPLIT | Split Persons
- **Type / Role:** `splitOut`; splits person field into separate items.
- **Configuration:** `fieldToSplitOut = person`
- **Failure / edge cases:** If `person` is a scalar string rather than array-like, behavior may vary.

#### GS | Read Person Mapping
- **Type / Role:** `googleSheets`; loads alias mapping.
- **Expected columns:** `find`, `replace`
- **Output:** `MERGE | Join Persons with Mapping`

#### MERGE | Join Persons with Mapping
- **Type / Role:** `merge` with SQL.
- **Configuration:** `SELECT * FROM input1 LEFT JOIN input2 ON input1.category = input2.find`
- **Important issue:** This query joins on `input1.category`, but this is the person flow. It likely should join on `input1.person`. As configured, it appears incorrect.
- **Output:** `SET | Normalize Person`
- **Failure / edge cases:** Person alias resolution may fail because of this likely misconfiguration.

#### SET | Normalize Person
- **Type / Role:** `set`; uses mapped person if found, otherwise original.
- **Configuration:** `person_new = replace ?? person`
- **Output:** `MERGE | Check Person against Allowed`

#### GS | Read Allowed Persons
- **Type / Role:** `googleSheets`; loads person master list.
- **Configuration:** Reads tab `list_persons`.
- **Expected columns:** `person`
- **Outputs:** To validation merge and person list aggregate.
- **Failure / edge cases:** Spreadsheet tab must exist; sheetName uses numeric gid.

#### MERGE | Check Person against Allowed
- **Type / Role:** `merge` with SQL; validates `person_new = person`.
- **Output:** `IF | Person Known?`

#### IF | Person Known?
- **Type / Role:** `if`; checks whether master-list match exists.
- **Configuration:** Tests whether `description` does not exist.
- **Behavior:** Unknown values go to classification path; known values continue directly.
- **Output:** To `MERGE | Combine Persons with List` and `MERGE | Loop Entry (Person)`

#### AGG | Aggregate Person List
- **Type / Role:** `aggregate`; creates array of allowed persons.
- **Output:** `MERGE | Combine Persons with List`

#### MERGE | Combine Persons with List
- **Type / Role:** `merge`; combines unresolved person with allowed list.
- **Output:** `LOOP | Iterate Persons`

#### LOOP | Iterate Persons
- **Type / Role:** `splitInBatches`; iterates unresolved persons one by one.
- **Outputs:** Loop continuation and classification path.

#### LLM | Classify Person
- **Type / Role:** OpenAI node; maps unknown person to allowed list.
- **Configuration:** Strict one-name output from allowed person list.
- **Output:** `SET | Extract LLM Person`
- **Failure / edge cases:** Non-exact outputs or semantic mismatches.

#### SET | Extract LLM Person
- **Type / Role:** `set` raw mode.
- **Configuration:** `jsonOutput = {{ $json.message }}`
- **Output:** `TG | Confirm Person Suggestion`

#### TG | Confirm Person Suggestion
- **Type / Role:** `telegram`; asks user to confirm AI-suggested person.
- **Configuration:** `sendAndWait` with Yes/No.
- **Output:** `IF | Person Suggestion Accepted?`

#### IF | Person Suggestion Accepted?
- **Type / Role:** `if`; checks approval.
- **Outputs:**
  - True: `SET | Set New+Old Person (LLM)`
  - False: `JS | Build Person Inline Buttons`

#### JS | Build Person Inline Buttons
- **Type / Role:** `code`; creates button list from allowed persons.
- **Configuration:** Same resume-url mechanism as category picker.
- **Output:** `HTTP | Send Person Selection Message`

#### HTTP | Send Person Selection Message
- **Type / Role:** `httpRequest`; directly calls Telegram API.
- **Configuration:** Requires hardcoded bot token replacement in URL.
- **Output:** `WAIT | Wait for Person Selection`

#### WAIT | Wait for Person Selection
- **Type / Role:** `wait`; pauses until callback data is forwarded via resume webhook.
- **Output:** `SET | Read Callback Body (Person)`

#### SET | Read Callback Body (Person)
- **Type / Role:** `set` raw mode; unwraps resume request body.
- **Output:** `SET | Set New+Old Person (Selection)`

#### SET | Set New+Old Person (Selection)
- **Type / Role:** `set`; builds mapping from manual selection.
- **Output:** `MERGE | Combine Person Mapping Entries`

#### SET | Set New+Old Person (LLM)
- **Type / Role:** `set`; builds mapping from approved AI suggestion.
- **Output:** `MERGE | Combine Person Mapping Entries`

#### MERGE | Combine Person Mapping Entries
- **Type / Role:** `merge`; converges selection and AI-confirmed mapping paths.
- **Output:** `GS | Save Person Mapping`

#### GS | Save Person Mapping
- **Type / Role:** `googleSheets`; appends learned person alias mapping.
- **Configuration:** Appends `find = old`, `replace = new` to `person_mapping`.
- **Output:** `LOOP | Iterate Persons`

#### MERGE | Loop Entry (Person)
- **Type / Role:** `merge`; loop re-entry point.
- **Output:** `SET | Set Resolved Person`

#### SET | Set Resolved Person
- **Type / Role:** `set`; outputs final canonical person.
- **Configuration:** `person_new = replace ?? original person_new`
- **Output:** `MERGE | Combine Person Mapping`

#### MERGE | Combine Person Mapping
- **Type / Role:** `merge`; funnels person-resolution result toward final intent merge.
- **Output:** `MERGE | Combine Category + Person`

---

## 2.5 Intent Reassembly and Data Loading

### Overview
This block recombines parsed intent fields with any resolved person/category values, then reads the Google Sheets data needed for querying.

### Nodes Involved
- `MERGE | Combine Category + Person`
- `JS | Merge Intent Fields`
- `SET | Assemble Resolved Intent`
- `GS | Load Expenses`
- `GS | Load Categories`
- `MERGE | Combine Expenses + Categories`

### Node Details

#### MERGE | Combine Category + Person
- **Type / Role:** `merge`; receives up to three streams:
  - category result or skip path,
  - resolved categories,
  - resolved person mapping result.
- **Configuration:** `numberInputs = 3`
- **Output:** `JS | Merge Intent Fields`
- **Failure / edge cases:** If one branch emits nothing, merge behavior depends on execution timing and n8n merge semantics.

#### JS | Merge Intent Fields
- **Type / Role:** `code`; merges partial outputs into one object.
- **Configuration:** Iterates through all incoming items, keeping non-empty values.
- **Output:** `SET | Assemble Resolved Intent`
- **Failure / edge cases:**
  - Last non-empty value wins if there are conflicts.
  - Boolean false could be dropped if treated as empty in future changes, though current condition preserves false.

#### SET | Assemble Resolved Intent
- **Type / Role:** `set`; rebuilds full intent object.
- **Configuration:**
  - Copies base fields from `JS | Extract Intent JSON`
  - overwrites person with `person_new`
  - overwrites category with `category_new`
- **Important issue:** `common_only` is assigned as string type, even though downstream logic expects boolean/null.
- **Output:** Parallel to `GS | Load Expenses` and `GS | Load Categories`

#### GS | Load Expenses
- **Type / Role:** `googleSheets`; loads all expense rows.
- **Expected columns:** `date`, `amount`, `category`, `description`, `common_expense`, `Person`
- **Output:** `MERGE | Combine Expenses + Categories`
- **Failure / edge cases:** Missing required columns, non-numeric amount values, date formatting inconsistencies.

#### GS | Load Categories
- **Type / Role:** `googleSheets`; loads category reference data in parallel.
- **Output:** `MERGE | Combine Expenses + Categories`
- **Note:** The downstream code node does not actually use this dataset directly.
- **Failure / edge cases:** Mostly sheet connectivity issues.

#### MERGE | Combine Expenses + Categories
- **Type / Role:** `merge`; combines parallel sheet loads.
- **Configuration:** `chooseBranch`
- **Output:** `JS | Filter & Aggregate`
- **Important issue:** With `chooseBranch`, one branch may be preferred; this node may simply enforce synchronization rather than true data combination. Since downstream code fetches sheet data by node name directly, this merge is largely structural.

---

## 2.6 Query Execution and Aggregation

### Overview
This block performs the actual analytics. It resolves relative dates, filters expense rows, computes totals and counts, and optionally creates grouped breakdowns.

### Nodes Involved
- `JS | Filter & Aggregate`

### Node Details

#### JS | Filter & Aggregate
- **Type / Role:** `code`; core analytics engine.
- **Configuration highlights:**
  - Resolves `this_week`, `last_week`, `this_month`, `last_month`, `this_year`, `last_year`.
  - Uses explicit dates when provided.
  - Loads expenses by referencing `GS | Load Expenses`.
  - Overwrites intent `person` and `category` with resolved values from `SET | Assemble Resolved Intent`.
  - Filters by:
    - valid date,
    - date range,
    - person,
    - categories,
    - `common_only`.
  - Computes:
    - `total`
    - `count`
    - optional `categoryBreakdown`
    - optional `personBreakdown`
- **Key expressions / variables:**
  - `$('JS | Extract Intent JSON').first().json`
  - `$('SET | Assemble Resolved Intent').first().json`
  - `$('GS | Load Expenses').all()`
- **Outputs:** `JS | Format Response Message`
- **Failure / edge cases:**
  - `common_only` may arrive as string instead of boolean because of upstream `Set` typing.
  - `Person` column in expenses is capitalized and specifically referenced.
  - Dates parsed with JavaScript `Date`; locale-specific formats may fail.
  - Timezone handling may shift dates if input is not ISO-like.
  - Comparison intent is parsed but not implemented in analytics output.
  - `GS | Load Categories` is loaded but unused here.

---

## 2.7 Response Formatting and Delivery

### Overview
This block turns the computed result into a Markdown message and sends it back to Telegram as a reply to the original user message.

### Nodes Involved
- `JS | Format Response Message`
- `TG | Send Reply`

### Node Details

#### JS | Format Response Message
- **Type / Role:** `code`; builds final human-readable response text.
- **Configuration:**
  - Shows title, period, filters, total, transaction count.
  - If grouped by category or person, sorts descending and shows percentages.
  - Adds empty-state note when no results were found.
- **Important issue:** Breakdown lines use `$` while total uses `€`, so currency formatting is inconsistent.
- **Output:** `TG | Send Reply`
- **Failure / edge cases:**
  - If `filters.category` is an array, string interpolation may render comma-separated values.
  - `data.comparison` is checked, but upstream analytics does not populate it.

#### TG | Send Reply
- **Type / Role:** `telegram`; sends final response.
- **Configuration:**
  - `parse_mode = Markdown`
  - replies to original message ID
  - disables attribution append
- **Key expressions / variables:**
  - Chat ID from original inbound message
  - Reply-to message ID from original message
  - Text from `JS | Format Response Message`
- **Failure / edge cases:**
  - Markdown escaping issues if category/person values contain special characters.
  - If execution originated from callback branch, original `message` references could be missing, but this node is only used in the message path.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| MSG \| Telegram Inbound | telegramTrigger | Receives Telegram messages and callback queries |  | IF \| Message or Callback? | ## 🟢 LAYER 1 — INPUT<br>Receives incoming Telegram messages & callbacks.<br>Authorizes users and routes by message type.<br><br>---<br><br>⚙️ **ACTION REQUIRED**<br><br>**1. Connect Telegram Credential**<br>Open the `MSG \| Telegram Inbound` node and connect your Telegram Bot credential.<br>Do the same for all other `TG \|` nodes in this layer.<br>→ *Nodes: MSG \| Telegram Inbound, TG \| Confirm Category Selection, TG \| Confirm Person Selection*<br><br>**2. Add authorized Chat IDs**<br>Open `IF \| User Authorized?` and replace the placeholder values (`1000000000`, `1000000001`) with the real Telegram Chat IDs of all users who should have access.<br>Each person can find their Chat ID by messaging @userinfobot on Telegram.<br><br>**Purpose:** Entry point of the workflow. Receives all incoming Telegram events and routes them to the correct processing path.<br><br>**What happens here:**<br>- `MSG \| Telegram Inbound` — Listens for incoming Telegram messages and callback queries (inline button taps).<br>- `IF \| Message or Callback?` — Splits the flow: regular text messages go to intent parsing, callback queries (button responses) are handled separately.<br>- `IF \| User Authorized?` — Checks whether the sender's chat ID is on the allowlist. Unauthorized users are silently dropped.<br>- `IF \| Category or Person Callback?` — Determines whether an incoming callback is a category or person selection and routes accordingly.<br>- `JS \| Read Category Callback` / `JS \| Read Person Callback` — Extracts the selected value and the resume URL from the callback payload.<br>- `HTTP \| Forward Category/Person Selection` — Resumes the waiting workflow branch by POSTing the selected value to the internal webhook.<br>- `TG \| Confirm Category/Person Selection` — Sends a short confirmation message to the user acknowledging their selection. |
| IF \| Message or Callback? | if | Routes inbound event into message flow or callback flow | MSG \| Telegram Inbound | IF \| User Authorized?; IF \| Category or Person Callback? | same as above |
| IF \| User Authorized? | if | Restricts access to approved Telegram chat IDs | IF \| Message or Callback? | LLM \| Parse Intent | same as above |
| IF \| Category or Person Callback? | if | Routes callback selection to category or person resume flow | IF \| Message or Callback? | JS \| Read Category Callback; JS \| Read Person Callback | same as above |
| JS \| Read Category Callback | code | Extracts selected category and resume URL from callback | IF \| Category or Person Callback? | HTTP \| Forward Category Selection | same as above |
| HTTP \| Forward Category Selection | httpRequest | POSTs category selection to wait resume webhook | JS \| Read Category Callback | TG \| Confirm Category Selection | same as above |
| TG \| Confirm Category Selection | telegram | Confirms category selection to user | HTTP \| Forward Category Selection |  | same as above |
| JS \| Read Person Callback | code | Extracts selected person and resume URL from callback | IF \| Category or Person Callback? | HTTP \| Forward Person Selection | same as above |
| HTTP \| Forward Person Selection | httpRequest | POSTs person selection to wait resume webhook | JS \| Read Person Callback | TG \| Confirm Person Selection | same as above |
| TG \| Confirm Person Selection | telegram | Confirms person selection to user | HTTP \| Forward Person Selection |  | same as above |
| LLM \| Parse Intent | @n8n/n8n-nodes-langchain.openAi | Converts free-text query into structured JSON intent | IF \| User Authorized? | JS \| Extract Intent JSON | ## 🔵 LAYER 2 — INTENT PARSING<br><br>---<br><br>⚙️ **ACTION REQUIRED**<br><br>**1. Connect OpenAI Credential**<br>Open the `LLM \| Parse Intent` node and connect your OpenAI API credential.<br><br>**Purpose:** Converts the user's free-text message into a structured JSON object that all downstream layers can work with.<br><br>**What happens here:**<br>- `LLM \| Parse Intent` — Sends the raw message to GPT-4.1-nano with a strict system prompt. Returns a JSON object containing: intent, time_reference, explicit dates, person, category, common_only flag, comparison type and group_by dimension.<br>- `JS \| Extract Intent JSON` — Cleans the raw LLM response (strips markdown fences if present) and parses it into a proper JSON node output. Falls back to a safe empty-intent object if parsing fails. |
| JS \| Extract Intent JSON | code | Parses LLM output into JSON with fallback | LLM \| Parse Intent | IF \| Category Present?; IF \| Person Present? | same as above |
| IF \| Category Present? | if | Decides whether category resolution is needed | JS \| Extract Intent JSON | MERGE \| Combine Category + Person; SPLIT \| Split Categories; GS \| Read Category Mapping; GS \| Read Allowed Categories | ## 🟡 LAYER 3a — ENTITY RESOLUTION: CATEGORIES<br>Resolves raw category strings to canonical names via mapping sheets.<br>Unknown entities → LLM classification → user confirmation → alias saved.<br><br>---<br><br>⚙️ **ACTION REQUIRED**<br><br>**1. Connect your `categories_mapping` Google Sheet**<br>Open `GS \| Read Category Mapping` and `GS \| Save Category Mapping`.<br>Replace `YOUR_SPREADSHEET_ID` with the ID of your `categories_mapping` sheet.<br>Required columns: `find` · `replace`<br><br>**2. Connect your `expense_categories` Google Sheet**<br>Open `GS \| Read Allowed Categories`.<br>Replace `YOUR_SPREADSHEET_ID` with the ID of your `expense_categories` sheet.<br>Required columns: `category` · `description` · `examples`<br><br>**3. Connect OpenAI Credential**<br>Open `LLM \| Classify Category` and connect your OpenAI API credential.<br><br>**4. Connect Telegram Credential**<br>Open `TG \| Confirm Category Suggestion` and connect your Telegram Bot credential.<br><br>**5. Set your Bot Token in HTTP nodes**<br>Open `HTTP \| Send Category Selection Message`.<br>Replace `{{YOUR_BOT_TOKEN}}` in the URL with your actual Telegram Bot Token.<br><br>**Purpose:** Resolves raw category strings from the intent into valid canonical category names as defined in the categories master sheet.<br><br>**What happens here:**<br>- `IF \| Category Present?` — Skips the entire branch if no category was mentioned.<br>- `SPLIT \| Split Categories` — Splits multi-category intents into individual items for per-item processing.<br>- `GS \| Read Category Mapping` — Loads the alias→canonical mapping sheet (e.g. "essen" → "food").<br>- `MERGE \| Join Categories with Mapping` + `SET \| Normalize Category` — Applies alias replacement where available, keeps original otherwise.<br>- `GS \| Read Allowed Categories` + `MERGE \| Check Category against Allowed` — Validates the normalised category against the master list.<br>- `IF \| Category Known?` — Known categories pass through directly; unknown ones enter the LLM resolution loop.<br>- `LOOP \| Iterate Categories` — Processes unresolved categories one by one.<br>- `LLM \| Classify Category` — GPT maps the unknown input to the closest canonical category from the allowed list.<br>- `TG \| Confirm Category Suggestion` — Sends the LLM suggestion to the user as a Yes/No confirmation.<br>- `IF \| Category Suggestion Accepted?` — On confirmation the mapping is saved; otherwise an inline-button picker is shown (`JS \| Build Category Inline Buttons` → `HTTP \| Send Category Selection Message` → `WAIT \| Wait for Category Selection`).<br>- `GS \| Save Category Mapping` — Persists the newly learned alias for future queries. |
| SPLIT \| Split Categories | splitOut | Splits category array into one item per category | IF \| Category Present? | MERGE \| Join Categories with Mapping | same as above |
| GS \| Read Category Mapping | googleSheets | Loads category alias mapping sheet | IF \| Category Present? | MERGE \| Join Categories with Mapping | same as above |
| MERGE \| Join Categories with Mapping | merge | Left-joins raw categories with alias mappings | SPLIT \| Split Categories; GS \| Read Category Mapping | SET \| Normalize Category | same as above |
| SET \| Normalize Category | set | Chooses mapped category or original category | MERGE \| Join Categories with Mapping | MERGE \| Check Category against Allowed | same as above |
| GS \| Read Allowed Categories | googleSheets | Loads canonical category list | IF \| Category Present? | MERGE \| Check Category against Allowed; AGG \| Aggregate Category List | same as above |
| MERGE \| Check Category against Allowed | merge | Validates normalized category against master list | SET \| Normalize Category; GS \| Read Allowed Categories | IF \| Category Known? | same as above |
| IF \| Category Known? | if | Branches between known category flow and unresolved category flow | MERGE \| Check Category against Allowed | MERGE \| Combine Categories with List; MERGE \| Loop Entry (Category) | same as above |
| AGG \| Aggregate Category List | aggregate | Builds array of allowed categories | GS \| Read Allowed Categories | MERGE \| Combine Categories with List | same as above |
| MERGE \| Combine Categories with List | merge | Combines unresolved category items with full allowed list | IF \| Category Known?; AGG \| Aggregate Category List | LOOP \| Iterate Categories | same as above |
| LOOP \| Iterate Categories | splitInBatches | Iterates unresolved category items one by one | MERGE \| Combine Categories with List; GS \| Save Category Mapping | MERGE \| Loop Entry (Category); LLM \| Classify Category | same as above |
| LLM \| Classify Category | @n8n/n8n-nodes-langchain.openAi | Maps unknown category to one allowed category | LOOP \| Iterate Categories | SET \| Extract LLM Category | same as above |
| SET \| Extract LLM Category | set | Reshapes category LLM response | LLM \| Classify Category | TG \| Confirm Category Suggestion | same as above |
| TG \| Confirm Category Suggestion | telegram | Asks user to confirm AI category suggestion | SET \| Extract LLM Category | IF \| Category Suggestion Accepted? | same as above |
| IF \| Category Suggestion Accepted? | if | Branches between approved suggestion and manual category picker | TG \| Confirm Category Suggestion | SET \| Set New+Old Category (LLM); JS \| Build Category Inline Buttons | same as above |
| JS \| Build Category Inline Buttons | code | Builds Telegram inline keyboard for manual category selection | IF \| Category Suggestion Accepted? | HTTP \| Send Category Selection Message | same as above |
| HTTP \| Send Category Selection Message | httpRequest | Sends Telegram inline keyboard with embedded resume URL | JS \| Build Category Inline Buttons | WAIT \| Wait for Category Selection | same as above |
| WAIT \| Wait for Category Selection | wait | Pauses execution until user selects category | HTTP \| Send Category Selection Message | SET \| Read Callback Body (Cat.) | same as above |
| SET \| Read Callback Body (Cat.) | set | Extracts wait resume payload body for category selection | WAIT \| Wait for Category Selection | SET \| Set New+Old Category (Selection) | same as above |
| SET \| Set New+Old Category (Selection) | set | Prepares old/new category mapping from manual selection | SET \| Read Callback Body (Cat.) | MERGE \| Combine Category Mapping Entries | same as above |
| SET \| Set New+Old Category (LLM) | set | Prepares old/new category mapping from approved AI suggestion | IF \| Category Suggestion Accepted? | MERGE \| Combine Category Mapping Entries | same as above |
| MERGE \| Combine Category Mapping Entries | merge | Converges LLM-approved and manually selected mappings | SET \| Set New+Old Category (Selection); SET \| Set New+Old Category (LLM) | GS \| Save Category Mapping | same as above |
| GS \| Save Category Mapping | googleSheets | Persists learned category alias mapping | MERGE \| Combine Category Mapping Entries | LOOP \| Iterate Categories | same as above |
| MERGE \| Loop Entry (Category) | merge | Re-entry point for category loop output | IF \| Category Known?; LOOP \| Iterate Categories | SET \| Set Resolved Category | same as above |
| SET \| Set Resolved Category | set | Emits final resolved category value | MERGE \| Loop Entry (Category) | AGG \| Aggregate Resolved Categories | same as above |
| AGG \| Aggregate Resolved Categories | aggregate | Rebuilds final resolved category array | SET \| Set Resolved Category | MERGE \| Combine Category + Person | same as above |
| IF \| Person Present? | if | Decides whether person resolution is needed | JS \| Extract Intent JSON | MERGE \| Combine Person Mapping; SPLIT \| Split Persons; GS \| Read Person Mapping; GS \| Read Allowed Persons | ## 🟡 LAYER 3b — ENTITY RESOLUTION: PERSONS<br><br><br>---<br><br>⚙️ **ACTION REQUIRED**<br><br>**1. Connect your `person_mapping` Google Sheet**<br>Open `GS \| Read Person Mapping` and `GS \| Save Person Mapping`.<br>Replace `YOUR_SPREADSHEET_ID` with the ID of your `person_mapping` sheet.<br>Required columns: `find` · `replace`<br><br>**2. Connect your `list_persons` sheet**<br>Open `GS \| Read Allowed Persons`.<br>This reads from the `list_persons` tab of the same `person_mapping` spreadsheet.<br>Required column: `person`<br><br>**3. Connect OpenAI Credential**<br>Open `LLM \| Classify Person` and connect your OpenAI API credential.<br><br>**4. Connect Telegram Credential**<br>Open `TG \| Confirm Person Suggestion` and connect your Telegram Bot credential.<br><br>**5. Set your Bot Token in HTTP nodes**<br>Open `HTTP \| Send Person Selection Message`.<br>Replace `{{YOUR_BOT_TOKEN}}` in the URL with your actual Telegram Bot Token.<br><br>**Purpose:** Mirrors Layer 3a but for person names — resolves nicknames and unknown names to canonical person records.<br><br>**What happens here:**<br>- `IF \| Person Present?` — Skips the branch entirely when no person was specified.<br>- `SPLIT \| Split Persons` — Splits the person field into individual items.<br>- `GS \| Read Person Mapping` — Loads the alias→canonical person mapping (e.g. "nickname" → "Full Name").<br>- `MERGE \| Join Persons with Mapping` + `SET \| Normalize Person` — Applies alias replacement where available.<br>- `GS \| Read Allowed Persons` + `MERGE \| Check Person against Allowed` — Validates the resolved name against the known persons list.<br>- `IF \| Person Known?` — Known persons pass through; unknown ones enter the LLM resolution loop.<br>- `LOOP \| Iterate Persons` — Processes unresolved person names one at a time.<br>- `LLM \| Classify Person` — GPT maps the unknown name to the nearest known person from the allowed list.<br>- `TG \| Confirm Person Suggestion` — Asks the user to confirm the suggestion via Yes/No button.<br>- `IF \| Person Suggestion Accepted?` — On confirmation the mapping is saved; otherwise a button picker is shown (`JS \| Build Person Inline Buttons` → `HTTP \| Send Person Selection Message` → `WAIT \| Wait for Person Selection`).<br>- `GS \| Save Person Mapping` — Stores the new alias for future use. |
| SPLIT \| Split Persons | splitOut | Splits person values into separate items | IF \| Person Present? | MERGE \| Join Persons with Mapping | same as above |
| GS \| Read Person Mapping | googleSheets | Loads person alias mapping | IF \| Person Present? | MERGE \| Join Persons with Mapping | same as above |
| MERGE \| Join Persons with Mapping | merge | Joins raw person items with alias mapping | SPLIT \| Split Persons; GS \| Read Person Mapping | SET \| Normalize Person | same as above |
| SET \| Normalize Person | set | Chooses mapped person or original person | MERGE \| Join Persons with Mapping | MERGE \| Check Person against Allowed | same as above |
| GS \| Read Allowed Persons | googleSheets | Loads canonical person list | IF \| Person Present? | MERGE \| Check Person against Allowed; AGG \| Aggregate Person List | same as above |
| MERGE \| Check Person against Allowed | merge | Validates normalized person against master list | SET \| Normalize Person; GS \| Read Allowed Persons | IF \| Person Known? | same as above |
| IF \| Person Known? | if | Branches between known person flow and unresolved person flow | MERGE \| Check Person against Allowed | MERGE \| Combine Persons with List; MERGE \| Loop Entry (Person) | same as above |
| AGG \| Aggregate Person List | aggregate | Builds array of allowed persons | GS \| Read Allowed Persons | MERGE \| Combine Persons with List | same as above |
| MERGE \| Combine Persons with List | merge | Combines unresolved persons with allowed person list | IF \| Person Known?; AGG \| Aggregate Person List | LOOP \| Iterate Persons | same as above |
| LOOP \| Iterate Persons | splitInBatches | Iterates unresolved person items one by one | MERGE \| Combine Persons with List; GS \| Save Person Mapping | MERGE \| Loop Entry (Person); LLM \| Classify Person | same as above |
| LLM \| Classify Person | @n8n/n8n-nodes-langchain.openAi | Maps unknown person to one allowed person | LOOP \| Iterate Persons | SET \| Extract LLM Person | same as above |
| SET \| Extract LLM Person | set | Reshapes person LLM response | LLM \| Classify Person | TG \| Confirm Person Suggestion | same as above |
| TG \| Confirm Person Suggestion | telegram | Asks user to confirm AI person suggestion | SET \| Extract LLM Person | IF \| Person Suggestion Accepted? | same as above |
| IF \| Person Suggestion Accepted? | if | Branches between approved suggestion and manual person picker | TG \| Confirm Person Suggestion | SET \| Set New+Old Person (LLM); JS \| Build Person Inline Buttons | same as above |
| JS \| Build Person Inline Buttons | code | Builds Telegram inline keyboard for manual person selection | IF \| Person Suggestion Accepted? | HTTP \| Send Person Selection Message | same as above |
| HTTP \| Send Person Selection Message | httpRequest | Sends Telegram inline keyboard with embedded resume URL for persons | JS \| Build Person Inline Buttons | WAIT \| Wait for Person Selection | same as above |
| WAIT \| Wait for Person Selection | wait | Pauses execution until user selects person | HTTP \| Send Person Selection Message | SET \| Read Callback Body (Person) | same as above |
| SET \| Read Callback Body (Person) | set | Extracts wait resume payload body for person selection | WAIT \| Wait for Person Selection | SET \| Set New+Old Person (Selection) | same as above |
| SET \| Set New+Old Person (Selection) | set | Prepares old/new person mapping from manual selection | SET \| Read Callback Body (Person) | MERGE \| Combine Person Mapping Entries | same as above |
| SET \| Set New+Old Person (LLM) | set | Prepares old/new person mapping from approved AI suggestion | IF \| Person Suggestion Accepted? | MERGE \| Combine Person Mapping Entries | same as above |
| MERGE \| Combine Person Mapping Entries | merge | Converges approved/manual person mapping paths | SET \| Set New+Old Person (Selection); SET \| Set New+Old Person (LLM) | GS \| Save Person Mapping | same as above |
| GS \| Save Person Mapping | googleSheets | Persists learned person alias mapping | MERGE \| Combine Person Mapping Entries | LOOP \| Iterate Persons | same as above |
| MERGE \| Loop Entry (Person) | merge | Re-entry point for resolved person items | IF \| Person Known?; LOOP \| Iterate Persons | SET \| Set Resolved Person | same as above |
| SET \| Set Resolved Person | set | Emits final resolved person value | MERGE \| Loop Entry (Person) | MERGE \| Combine Person Mapping | same as above |
| MERGE \| Combine Person Mapping | merge | Funnels person resolution result to final intent merge | IF \| Person Present?; SET \| Set Resolved Person | MERGE \| Combine Category + Person | same as above |
| MERGE \| Combine Category + Person | merge | Combines category resolution and person resolution outputs | IF \| Category Present?; AGG \| Aggregate Resolved Categories; MERGE \| Combine Person Mapping | JS \| Merge Intent Fields | **Purpose:** Assembles the fully resolved intent and loads raw expense data from Google Sheets, ready for filtering.<br><br>**What happens here:**<br>- `SET \| Assemble Resolved Intent` — Merges canonical category and person values back into the original intent object, producing one clean query descriptor.<br>- `GS \| Load Expenses` — Reads all rows from the expenses Google Sheet (date, amount, category, person, common_expense flag).<br>- `GS \| Load Categories` — Loads the categories reference sheet in parallel (used for validation during filtering).<br>- `MERGE \| Combine Expenses + Categories` — Combines both data sources into a single output so the next layer has everything it needs. |
| JS \| Merge Intent Fields | code | Merges partial intent fragments into one object | MERGE \| Combine Category + Person | SET \| Assemble Resolved Intent | same as above |
| SET \| Assemble Resolved Intent | set | Rebuilds full intent using resolved person/category values | JS \| Merge Intent Fields | GS \| Load Expenses; GS \| Load Categories | same as above |
| GS \| Load Expenses | googleSheets | Loads expense transaction rows | SET \| Assemble Resolved Intent | MERGE \| Combine Expenses + Categories | ## 🟠 LAYER 4 — QUERY ENGINE<br>Loads expense data from Google Sheets.<br>Merges resolved intent with raw data for filtering.<br><br>---<br><br>⚙️ **ACTION REQUIRED**<br><br>**1. Connect your `expenses` Google Sheet**<br>Open `GS \| Load Expenses` and replace `YOUR_SPREADSHEET_ID` with your expenses sheet ID.<br>Required columns: `date` · `amount` · `category` · `description` · `common_expense` · `Person`<br><br>**2. Connect your `expense_categories` Google Sheet**<br>Open `GS \| Load Categories` and replace `YOUR_SPREADSHEET_ID` with your categories sheet ID.<br>Required columns: `category` · `description` · `examples`<br><br>same Layer 4 doc note above |
| GS \| Load Categories | googleSheets | Loads category reference rows in parallel | SET \| Assemble Resolved Intent | MERGE \| Combine Expenses + Categories | same as above |
| MERGE \| Combine Expenses + Categories | merge | Synchronizes parallel sheet loads before analytics | GS \| Load Expenses; GS \| Load Categories | JS \| Filter & Aggregate | same as above |
| JS \| Filter & Aggregate | code | Resolves dates, filters expense rows, computes totals and breakdowns | MERGE \| Combine Expenses + Categories | JS \| Format Response Message | ## 🔴 LAYER 5 — ANALYTICS & AGGREGATION<br>Applies date, person, category and common_only filters.<br>Computes totals, category breakdown and person breakdown.<br><br>---<br><br>⚙️ **ACTION REQUIRED**<br><br>No setup required for this layer.<br>The `JS \| Filter & Aggregate` node works automatically based on the resolved intent from the previous layers.<br>If you add custom columns to your expenses sheet, update the filter logic in this node accordingly.<br><br>**Purpose:** Applies all filters from the resolved intent to the expense data and computes the requested aggregations.<br><br>**What happens here:**<br>- `JS \| Filter & Aggregate` — Single code node that does everything: resolves relative date references (this_week, last_month, …) into concrete ranges, applies person / category / common_only filters row by row, then computes total amount, transaction count, and optional breakdowns by category or person (driven by the group_by field in the intent). |
| JS \| Format Response Message | code | Builds final Telegram response text | JS \| Filter & Aggregate | TG \| Send Reply | ## 🟣 LAYER 6 — RESPONSE<br>Formats the final message and sends it back via Telegram.<br><br>---<br><br>⚙️ **ACTION REQUIRED**<br><br>**1. Connect Telegram Credential**<br>Open `TG \| Send Reply` and connect your Telegram Bot credential.<br><br>**2. Adjust the message format (optional)**<br>Open `JS \| Format Response Message` to customise the output format, currency symbol, or language of the reply message.<br><br>**Purpose:** Formats the aggregation result into a human-readable Telegram message and delivers it back to the user.<br><br>**What happens here:**<br>- `JS \| Format Response Message` — Builds the final message string: period header, applied filters, total amount, transaction count, and — if requested — a sorted category or person breakdown with percentage shares. Handles the empty-result case gracefully.<br>- `TG \| Send Reply` — Sends the formatted message as a Markdown reply to the original chat, referencing the triggering message ID so it appears as a thread reply. |
| TG \| Send Reply | telegram | Sends final Markdown reply to Telegram | JS \| Format Response Message |  | same as above |
| StickyNote_Layer1 | stickyNote | Documentation/comment node |  |  |  |
| StickyNote_Layer2 | stickyNote | Documentation/comment node |  |  |  |
| StickyNote_Layer3 | stickyNote | Documentation/comment node |  |  |  |
| StickyNote_Layer4 | stickyNote | Documentation/comment node |  |  |  |
| StickyNote_Layer5 | stickyNote | Documentation/comment node |  |  |  |
| StickyNote_Layer6 | stickyNote | Documentation/comment node |  |  |  |
| StickyNote_Layer | stickyNote | Documentation/comment node |  |  |  |
| DOC \| Layer 1 — Input | stickyNote | Documentation/comment node |  |  |  |
| DOC \| Layer 2 — Intent Parsing | stickyNote | Documentation/comment node |  |  |  |
| DOC \| Layer 3a — Entity Resolution: Categories | stickyNote | Documentation/comment node |  |  |  |
| DOC \| Layer 3b — Entity Resolution: Persons | stickyNote | Documentation/comment node |  |  |  |
| DOC \| Layer 4 — Query Engine | stickyNote | Documentation/comment node |  |  |  |
| DOC \| Layer 5 — Analytics & Aggregation | stickyNote | Documentation/comment node |  |  |  |
| DOC \| Layer 6 — Response | stickyNote | Documentation/comment node |  |  |  |
| Sticky Note | stickyNote | Documentation/comment node |  |  |  |
| Sticky Note1 | stickyNote | Documentation/comment node |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it something like: `Telegram Expense Tracker — Query Bot`.

2. **Add Telegram trigger node**
   - Type: `Telegram Trigger`
   - Name: `MSG | Telegram Inbound`
   - Updates: `message`, `callback_query`
   - Connect Telegram Bot credential.

3. **Add message/callback router**
   - Type: `IF`
   - Name: `IF | Message or Callback?`
   - Condition: check whether `message` exists.
   - Connect from `MSG | Telegram Inbound`.

4. **Add authorization node**
   - Type: `IF`
   - Name: `IF | User Authorized?`
   - Add OR conditions comparing `{{$json.message.chat.id}}` to each allowed chat ID.
   - Connect from true/message branch of `IF | Message or Callback?`.

5. **Add callback-type router**
   - Type: `IF`
   - Name: `IF | Category or Person Callback?`
   - Set condition to identify category-selection callback messages. In the original workflow it checks whether the callback message text contains `Kategorie`.
   - Connect from false/callback branch of `IF | Message or Callback?`.

6. **Build callback extractor for category**
   - Type: `Code`
   - Name: `JS | Read Category Callback`
   - Read:
     - `callback_query.data` as selected category
     - `callback_query.message.entities` to find the embedded `text_link`
     - extract `resumeUrl`
   - Output JSON with `category` and `resumeUrl`.

7. **Add category selection forwarder**
   - Type: `HTTP Request`
   - Name: `HTTP | Forward Category Selection`
   - Method: `POST`
   - URL: `{{$json.resumeUrl}}`
   - JSON body: `{ "category": "..." }`
   - Connect from `JS | Read Category Callback`.

8. **Add category confirmation sender**
   - Type: `Telegram`
   - Name: `TG | Confirm Category Selection`
   - Send message to callback chat confirming chosen category.
   - Connect Telegram credential.
   - Connect from `HTTP | Forward Category Selection`.

9. **Build callback extractor for person**
   - Type: `Code`
   - Name: `JS | Read Person Callback`
   - Same pattern as category callback extractor.
   - Output JSON with `person` and `resumeUrl`.

10. **Add person selection forwarder**
    - Type: `HTTP Request`
    - Name: `HTTP | Forward Person Selection`
    - Method: `POST`
    - URL: `{{$json.resumeUrl}}`
    - JSON body: `{ "person": "..." }`

11. **Add person confirmation sender**
    - Type: `Telegram`
    - Name: `TG | Confirm Person Selection`
    - Send confirmation to callback chat.
    - Connect Telegram credential.

12. **Add OpenAI intent parser**
    - Type: `OpenAI` (LangChain node)
    - Name: `LLM | Parse Intent`
    - Model: `gpt-4.1-nano`
    - Add:
      - one large system prompt defining JSON-only output and parsing rules,
      - one user message containing `{{$json.message.text}}`
    - Connect OpenAI credential.
    - Connect from authorized branch.

13. **Add intent JSON parser**
    - Type: `Code`
    - Name: `JS | Extract Intent JSON`
    - Logic:
      - read model content,
      - strip code fences,
      - parse JSON,
      - fallback to default empty intent object.
    - Connect from `LLM | Parse Intent`.

14. **Create category-presence check**
    - Type: `IF`
    - Name: `IF | Category Present?`
    - Test whether parsed category exists.
    - One branch should skip category resolution; the other should process categories.

15. **Create person-presence check**
    - Type: `IF`
    - Name: `IF | Person Present?`
    - Test whether parsed person exists.
    - One branch should skip person resolution; the other should process persons.

16. **Create category split node**
    - Type: `Split Out`
    - Name: `SPLIT | Split Categories`
    - Field to split: `category`

17. **Add category mapping sheet reader**
    - Type: `Google Sheets`
    - Name: `GS | Read Category Mapping`
    - Operation: read rows
    - Spreadsheet: `categories_mapping`
    - Required columns: `find`, `replace`
    - Connect Google Sheets OAuth2 credential.

18. **Add category join node**
    - Type: `Merge`
    - Name: `MERGE | Join Categories with Mapping`
    - Mode: SQL / combine by SQL
    - Query: left join split category items with mapping rows on category = find.

19. **Add category normalization node**
    - Type: `Set`
    - Name: `SET | Normalize Category`
    - Set `category_new` to mapped `replace` if present, else original `category`.

20. **Add allowed category sheet reader**
    - Type: `Google Sheets`
    - Name: `GS | Read Allowed Categories`
    - Spreadsheet: `expense_categories`
    - Required columns: `category`, `description`, `examples`

21. **Add category validation merge**
    - Type: `Merge`
    - Name: `MERGE | Check Category against Allowed`
    - SQL join on `category_new = category`

22. **Add known-category check**
    - Type: `IF`
    - Name: `IF | Category Known?`
    - Detect whether allowed category join matched.

23. **Add category list aggregate**
    - Type: `Aggregate`
    - Name: `AGG | Aggregate Category List`
    - Aggregate field: `category`

24. **Add combine-with-category-list node**
    - Type: `Merge`
    - Name: `MERGE | Combine Categories with List`
    - Combine unresolved item with aggregate list.

25. **Add category iteration loop**
    - Type: `Split In Batches`
    - Name: `LOOP | Iterate Categories`

26. **Add category classifier**
    - Type: `OpenAI` (LangChain)
    - Name: `LLM | Classify Category`
    - Model: `gpt-4.1-nano`
    - Prompt: strictly choose exactly one category from allowed list.
    - Connect OpenAI credential.

27. **Add category LLM extraction node**
    - Type: `Set`
    - Name: `SET | Extract LLM Category`
    - Raw JSON output from LLM response.

28. **Add category suggestion confirmation**
    - Type: `Telegram`
    - Name: `TG | Confirm Category Suggestion`
    - Operation: `sendAndWait`
    - Approval type: double
    - Approve label: `✅ Yes`
    - Disapprove label: `❌ No`
    - Message asks whether suggested category is correct.
    - Connect Telegram credential.

29. **Add category approval check**
    - Type: `IF`
    - Name: `IF | Category Suggestion Accepted?`
    - Condition: `{{$json.data.approved}} === true`

30. **Add category inline keyboard builder**
    - Type: `Code`
    - Name: `JS | Build Category Inline Buttons`
    - Build `inline_keyboard` from allowed categories.
    - Also output `$execution.resumeUrl`.

31. **Add direct Telegram API sender for category picker**
    - Type: `HTTP Request`
    - Name: `HTTP | Send Category Selection Message`
    - Method: `POST`
    - URL: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage`
    - Body:
      - chat_id
      - HTML text containing hidden link with resume URL
      - inline keyboard reply markup
    - Replace bot token manually.

32. **Add category wait node**
    - Type: `Wait`
    - Name: `WAIT | Wait for Category Selection`
    - Resume mode: `Webhook`
    - Method: `POST`

33. **Add category callback-body extractor**
    - Type: `Set`
    - Name: `SET | Read Callback Body (Cat.)`
    - Raw JSON output: request body.

34. **Add category mapping set nodes**
    - `SET | Set New+Old Category (Selection)` for manual picker result
    - `SET | Set New+Old Category (LLM)` for approved AI result
    - Each should output:
      - `old` = original unknown category
      - `new` = selected/approved canonical category

35. **Add category mapping merge**
    - Type: `Merge`
    - Name: `MERGE | Combine Category Mapping Entries`

36. **Add category mapping saver**
    - Type: `Google Sheets`
    - Name: `GS | Save Category Mapping`
    - Operation: append
    - Spreadsheet: `categories_mapping`
    - Write columns `find` and `replace`

37. **Add category loop re-entry**
    - Type: `Merge`
    - Name: `MERGE | Loop Entry (Category)`

38. **Add resolved category setter**
    - Type: `Set`
    - Name: `SET | Set Resolved Category`
    - Set `category_new` to final canonical category.

39. **Add resolved category aggregate**
    - Type: `Aggregate`
    - Name: `AGG | Aggregate Resolved Categories`
    - Aggregate `category_new`.

40. **Create person split node**
    - Type: `Split Out`
    - Name: `SPLIT | Split Persons`
    - Field: `person`

41. **Add person mapping sheet reader**
    - Type: `Google Sheets`
    - Name: `GS | Read Person Mapping`
    - Spreadsheet: `person_mapping`
    - Required columns: `find`, `replace`

42. **Add person join node**
    - Type: `Merge`
    - Name: `MERGE | Join Persons with Mapping`
    - Intended SQL: join person items with mapping table on `person = find`
    - Note: the exported workflow appears to use `input1.category`, which should be corrected when rebuilding.

43. **Add person normalization**
    - Type: `Set`
    - Name: `SET | Normalize Person`
    - Set `person_new = replace ?? person`

44. **Add allowed person sheet reader**
    - Type: `Google Sheets`
    - Name: `GS | Read Allowed Persons`
    - Read tab `list_persons`
    - Required column: `person`

45. **Add person validation merge**
    - Type: `Merge`
    - Name: `MERGE | Check Person against Allowed`
    - SQL join on `person_new = person`

46. **Add known-person check**
    - Type: `IF`
    - Name: `IF | Person Known?`

47. **Add person list aggregate**
    - Type: `Aggregate`
    - Name: `AGG | Aggregate Person List`
    - Aggregate field `person`

48. **Add combine-with-person-list node**
    - Type: `Merge`
    - Name: `MERGE | Combine Persons with List`

49. **Add person loop**
    - Type: `Split In Batches`
    - Name: `LOOP | Iterate Persons`

50. **Add person classifier**
    - Type: `OpenAI`
    - Name: `LLM | Classify Person`
    - Model: `gpt-4.1-nano`
    - Strictly pick one allowed person.

51. **Add person LLM extraction**
    - Type: `Set`
    - Name: `SET | Extract LLM Person`

52. **Add person suggestion confirmation**
    - Type: `Telegram`
    - Name: `TG | Confirm Person Suggestion`
    - Operation: `sendAndWait`
    - Same approval setup as categories.

53. **Add person approval check**
    - Type: `IF`
    - Name: `IF | Person Suggestion Accepted?`

54. **Add person inline keyboard builder**
    - Type: `Code`
    - Name: `JS | Build Person Inline Buttons`
    - Build `inline_keyboard` and expose `$execution.resumeUrl`.

55. **Add direct Telegram API sender for person picker**
    - Type: `HTTP Request`
    - Name: `HTTP | Send Person Selection Message`
    - URL: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage`

56. **Add person wait node**
    - Type: `Wait`
    - Name: `WAIT | Wait for Person Selection`
    - Resume: `webhook`
    - Method: `POST`

57. **Add person callback-body extractor**
    - Type: `Set`
    - Name: `SET | Read Callback Body (Person)`

58. **Add person old/new mapping setters**
    - `SET | Set New+Old Person (Selection)`
    - `SET | Set New+Old Person (LLM)`

59. **Add person mapping merge**
    - Type: `Merge`
    - Name: `MERGE | Combine Person Mapping Entries`

60. **Add person mapping saver**
    - Type: `Google Sheets`
    - Name: `GS | Save Person Mapping`
    - Append columns `find`, `replace` to `person_mapping`

61. **Add person loop re-entry**
    - Type: `Merge`
    - Name: `MERGE | Loop Entry (Person)`

62. **Add resolved person setter**
    - Type: `Set`
    - Name: `SET | Set Resolved Person`

63. **Add person result funnel**
    - Type: `Merge`
    - Name: `MERGE | Combine Person Mapping`

64. **Add final category/person merge**
    - Type: `Merge`
    - Name: `MERGE | Combine Category + Person`
    - Number of inputs: 3

65. **Add intent field merger**
    - Type: `Code`
    - Name: `JS | Merge Intent Fields`
    - Merge non-empty fields from all incoming items into one object.

66. **Add resolved intent assembler**
    - Type: `Set`
    - Name: `SET | Assemble Resolved Intent`
    - Rebuild all intent fields:
      - intent
      - time_reference
      - explicit_start_date
      - explicit_end_date
      - person
      - category
      - common_only
      - comparison
      - group_by
    - Prefer resolved person/category values.
    - Recommended improvement: store `common_only` as boolean, not string.

67. **Add expenses sheet reader**
    - Type: `Google Sheets`
    - Name: `GS | Load Expenses`
    - Spreadsheet: `expenses`
    - Required columns:
      - `date`
      - `amount`
      - `category`
      - `description`
      - `common_expense`
      - `Person`

68. **Add categories sheet reader for query layer**
    - Type: `Google Sheets`
    - Name: `GS | Load Categories`
    - Spreadsheet: `expense_categories`

69. **Add sheet synchronization merge**
    - Type: `Merge`
    - Name: `MERGE | Combine Expenses + Categories`
    - Connect both sheet readers into it.

70. **Add analytics node**
    - Type: `Code`
    - Name: `JS | Filter & Aggregate`
    - Implement:
      - relative date calculation,
      - explicit date override,
      - expense normalization,
      - person/category/common filtering,
      - total/count,
      - category or person breakdown.

71. **Add formatter node**
    - Type: `Code`
    - Name: `JS | Format Response Message`
    - Build Telegram Markdown response.
    - Recommended improvement: use one consistent currency symbol.

72. **Add final Telegram send node**
    - Type: `Telegram`
    - Name: `TG | Send Reply`
    - Send text from formatter.
    - Chat ID: original message chat ID.
    - Reply to original message ID.
    - Parse mode: Markdown.
    - Connect Telegram credential.

73. **Connect all branches exactly**
    - Message path:
      - Trigger → Message/Callback IF → Authorized IF → Intent parser → JSON extractor
      - JSON extractor → category branch + person branch
      - resolved outputs → merge → intent assembly → sheet loads → analytics → formatter → reply
    - Callback path:
      - Trigger → Message/Callback IF → Category/Person callback IF → callback extractor → forward HTTP → confirmation
    - Category and person wait nodes must be resumed through the callback branch.

74. **Prepare credentials**
    - Telegram Bot credential for:
      - trigger
      - all `TG | ...` nodes
    - OpenAI credential for:
      - `LLM | Parse Intent`
      - `LLM | Classify Category`
      - `LLM | Classify Person`
    - Google Sheets OAuth2 credential for all sheet nodes.

75. **Prepare Google Sheets structure**
    - `expenses`
    - `expense_categories`
    - `categories_mapping`
    - `person_mapping`
    - `list_persons`

76. **Recommended corrections while rebuilding**
    - Fix `MERGE | Join Persons with Mapping` to join on person, not category.
    - Make `common_only` boolean in `SET | Assemble Resolved Intent`.
    - Align currency formatting in `JS | Format Response Message`.
    - Consider replacing language-dependent callback detection (`Kategorie`) with a more robust marker.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow answers natural-language expense questions in Telegram using AI and Google Sheets. | General project purpose |
| Example questions include: “How much did I spend on food last month?”, “What did we spend together this week?”, and “Show me a breakdown by category for January.” | Usage examples |
| Five Google Sheets are required: `expenses`, `expense_categories`, `categories_mapping`, `person_mapping`, `list_persons`. | Data prerequisites |
| `expenses` required columns: `date`, `amount`, `category`, `description`, `common_expense`, `Person`. | Main data sheet |
| `expense_categories` required columns: `category`, `description`, `examples`. | Allowed category list |
| `categories_mapping` required columns: `find`, `replace`. | Category alias learning table |
| `person_mapping` required columns: `find`, `replace`. | Person alias learning table |
| `list_persons` required columns include at least `person`; sticky note also mentions `description`. | Person reference list |
| Key features listed in the workflow notes: German and English queries, GPT-4.1-nano intent parsing, self-learning entity resolution, Telegram inline disambiguation, relative date support, group-by summaries, and shared-expense filtering. | Feature summary |
| Multi-user support depends on adding approved Telegram Chat IDs in `IF \| User Authorized?`. | Security / access setup |
| Users can discover their Telegram chat ID by messaging `@userinfobot` on Telegram. | Setup aid |
| Shared expenses are identified via the `common_expense` boolean column. | Query behavior |
| The workflow “learns over time” by saving approved category and person aliases back to Google Sheets. | Design note |