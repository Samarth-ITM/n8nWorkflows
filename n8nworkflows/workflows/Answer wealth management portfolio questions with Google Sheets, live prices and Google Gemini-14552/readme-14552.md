Answer wealth management portfolio questions with Google Sheets, live prices and Google Gemini

https://n8nworkflows.xyz/workflows/answer-wealth-management-portfolio-questions-with-google-sheets--live-prices-and-google-gemini-14552


# Answer wealth management portfolio questions with Google Sheets, live prices and Google Gemini

## 1. Workflow Overview

This workflow implements a chat-based portfolio question assistant for wealth management use cases. A user sends a message in the form `C001: Your question`, where the prefix is a client ID and the remainder is the portfolio-related question. The workflow validates the message, looks up the client and holdings in Google Sheets, fetches live market prices and optional market context from external APIs, computes portfolio metrics, asks Google Gemini to generate a concise answer constrained to the provided data, logs the interaction, optionally creates a Gmail draft, and returns a final response.

Typical use cases include:

- Instant portfolio status questions from advisors or client support teams
- Lightweight portfolio reporting based on spreadsheet-stored holdings
- AI-assisted but data-constrained responses using live prices
- Audit-friendly interaction logging

### 1.1 Input Reception and Validation
The workflow starts from a chat trigger, parses the incoming message, validates that it contains a client ID and question, and exits early on invalid input.

### 1.2 Client and Holdings Retrieval
If the message is valid, the workflow looks up the client profile and portfolio holdings in Google Sheets. It stops early if the client does not exist or if no holdings are found.

### 1.3 Live Price Retrieval and Portfolio Computation
The workflow prepares the asset symbols, fetches live prices from an external HTTP API, normalizes the response, and calculates holding-level and portfolio-level metrics such as invested amount, current value, P&L, and best/worst performer.

### 1.4 Market Context and AI Prompt Construction
It optionally fetches market summary data, merges it into the portfolio snapshot, and builds a tightly constrained prompt for Gemini.

### 1.5 AI Answer Generation, Output Formatting, and Logging
The workflow generates the AI response, extracts plain text, determines success/failure, logs the result to Google Sheets, optionally creates a Gmail draft, and returns the final output to the chat caller.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

**Overview:**  
This block receives the user’s chat message, extracts the client identifier and question, and validates the expected `CLIENT_ID: question` format. Invalid input is converted into a structured failure payload, logged, and returned immediately.

**Nodes Involved:**  
- When chat message received
- Parse Client Message
- IF Valid Input?
- Build Invalid Input Response
- Log Invalid Input
- Return Invalid Response

### Node Details

#### 1) When chat message received
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Entry-point node that listens for chat input.
- **Configuration choices:**  
  - Response mode is set to `lastNode`, so the final node output becomes the chat response.
- **Key expressions or variables used:**  
  - No custom expressions.
- **Input and output connections:**  
  - No input.
  - Outputs to `Parse Client Message`.
- **Version-specific requirements:**  
  - Uses node type version `1.4`.
- **Edge cases or potential failure types:**  
  - Chat trigger availability depends on n8n chat configuration.
  - Input payload shape may vary (`chatInput`, `message`, `text`), which is handled downstream.
- **Sub-workflow reference:**  
  - None.

#### 2) Parse Client Message
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses inbound text and performs first-stage validation.
- **Configuration choices:**  
  - Runs `runOnceForEachItem`.
  - Looks for text in:
    - `$json.chatInput`
    - `$json.message`
    - `$json.text`
  - Requires a colon `:` separator.
  - Splits the message into:
    - `client_id`
    - `question`
  - Normalizes `client_id` to uppercase.
- **Key expressions or variables used:**  
  - `text = $json.chatInput || $json.message || $json.text || ''`
  - `client_id = parts[0].trim().toUpperCase()`
  - `question = parts.slice(1).join(':').trim()`
- **Input and output connections:**  
  - Input from `When chat message received`
  - Output to `IF Valid Input?`
- **Version-specific requirements:**  
  - Code node version `2`.
- **Edge cases or potential failure types:**  
  - Empty input
  - Missing `:`
  - Missing client ID
  - Missing question
  - Unexpected non-string input still gets string-coerced
- **Sub-workflow reference:**  
  - None.

#### 3) IF Valid Input?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches execution based on whether parsing produced a valid request.
- **Configuration choices:**  
  - Checks whether `{{ $json.valid }}` is boolean `true`.
- **Key expressions or variables used:**  
  - `={{ $json.valid }}`
- **Input and output connections:**  
  - Input from `Parse Client Message`
  - True branch → `Get Client Profile`
  - False branch → `Build Invalid Input Response`
- **Version-specific requirements:**  
  - IF node version `2.2`
- **Edge cases or potential failure types:**  
  - Strict boolean validation means malformed `valid` values could fail branch logic if not true boolean.
- **Sub-workflow reference:**  
  - None.

#### 4) Build Invalid Input Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts validation failure into a consistent error payload for logging and reply.
- **Configuration choices:**  
  - Produces fields such as:
    - `status: failed`
    - `error_stage`
    - `action_taken`
    - `reply`
    - `timestamp`
- **Key expressions or variables used:**  
  - References current item fields such as `$json.error_stage`, `$json.error_message`, `$json.raw_message`
- **Input and output connections:**  
  - Input from false branch of `IF Valid Input?`
  - Output to `Log Invalid Input`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - If upstream omitted some fields, defaults are applied.
- **Sub-workflow reference:**  
  - None.

#### 5) Log Invalid Input
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends invalid interaction records to the logging sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet: `Portfolio_AI_System`
  - Sheet: `interaction_logs`
  - Uses explicit column mapping schema for:
    - timestamp
    - client_id
    - question
    - reply
    - status
    - action_taken
    - error_stage
    - source_channel
- **Key expressions or variables used:**  
  - Implicit mapping from incoming JSON fields.
- **Input and output connections:**  
  - Input from `Build Invalid Input Response`
  - Output to `Return Invalid Response`
- **Version-specific requirements:**  
  - Google Sheets node version `4.7`
  - Requires Google Sheets OAuth2 credential.
- **Edge cases or potential failure types:**  
  - OAuth expiration
  - Spreadsheet permission errors
  - Missing worksheet or renamed columns
- **Sub-workflow reference:**  
  - None.

#### 6) Return Invalid Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Reduces the logged invalid-input payload to a lightweight response for the caller.
- **Configuration choices:**  
  - Returns:
    - `reply`
    - `status`
    - `error_stage`
- **Key expressions or variables used:**  
  - `$json.reply`, `$json.status`, `$json.error_stage`
- **Input and output connections:**  
  - Input from `Log Invalid Input`
  - Final branch output, no downstream node
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - If logging fails, this node may never execute.
- **Sub-workflow reference:**  
  - None.

---

## 2.2 Client and Portfolio Lookup

**Overview:**  
This block retrieves the client profile and holdings from Google Sheets using the parsed `client_id`. It then prepares the symbol list used for market-price retrieval and short-circuits if the client or holdings are missing.

**Nodes Involved:**  
- Get Client Profile
- IF Client Found?
- Build Client Not Found Response
- Get Client Holdings
- Prepare Symbols
- IF Holdings Found?
- Build No Holdings Response

### Node Details

#### 7) Get Client Profile
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Looks up the client profile row by `client_id`.
- **Configuration choices:**  
  - Spreadsheet: `Portfolio_AI_System`
  - Sheet: `clients` (shown as `gid=0`)
  - Filter:
    - `lookupColumn = client_id`
    - `lookupValue = {{ $json.client_id }}`
- **Key expressions or variables used:**  
  - `={{ $json.client_id }}`
- **Input and output connections:**  
  - Input from true branch of `IF Valid Input?`
  - Output to `IF Client Found?`
- **Version-specific requirements:**  
  - Google Sheets node version `4.7`
- **Edge cases or potential failure types:**  
  - No matching row returns empty result
  - Duplicate client IDs could return multiple rows depending on sheet state
  - Auth/permission errors
- **Sub-workflow reference:**  
  - None.

#### 8) IF Client Found?
- **Type and technical role:** `n8n-nodes-base.if`  
  Confirms that the client lookup returned a row containing a `client_id`.
- **Configuration choices:**  
  - Condition: `{{ !!$json.client_id }}` is true
- **Key expressions or variables used:**  
  - `={{ !!$json.client_id }}`
- **Input and output connections:**  
  - Input from `Get Client Profile`
  - True branch → `Get Client Holdings`
  - False branch → `Build Client Not Found Response`
- **Version-specific requirements:**  
  - IF node version `2.2`
- **Edge cases or potential failure types:**  
  - If sheet columns differ from expected names, valid rows may look missing.
- **Sub-workflow reference:**  
  - None.

#### 9) Build Client Not Found Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Generates a clean failure payload when the client ID is absent from the sheet.
- **Configuration choices:**  
  - Reads original parsed request from `Parse Client Message`.
  - Returns a standardized failure object for downstream logging.
- **Key expressions or variables used:**  
  - `$('Parse Client Message').first().json`
- **Input and output connections:**  
  - Input from false branch of `IF Client Found?`
  - Output to `Log Result`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - Cross-node reference assumes `Parse Client Message` executed successfully.
- **Sub-workflow reference:**  
  - None.

#### 10) Get Client Holdings
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Fetches all holdings rows for the matched client.
- **Configuration choices:**  
  - Spreadsheet: `Portfolio_AI_System`
  - Sheet: `holdings`
  - Filter:
    - `lookupColumn = client_id`
    - `lookupValue = {{ $('Parse Client Message').first().json.client_id }}`
- **Key expressions or variables used:**  
  - `={{ $('Parse Client Message').first().json.client_id }}`
- **Input and output connections:**  
  - Input from true branch of `IF Client Found?`
  - Output to `Prepare Symbols`
- **Version-specific requirements:**  
  - Google Sheets node version `4.7`
- **Edge cases or potential failure types:**  
  - No matching holdings
  - Duplicate or malformed rows
  - Numeric values stored as text
- **Sub-workflow reference:**  
  - None.

#### 11) Prepare Symbols
- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes holdings and extracts stock symbols for pricing.
- **Configuration choices:**  
  - Collects all incoming holdings rows.
  - Filters out empty objects.
  - If no holdings exist, returns `holdings_found: false`.
  - Otherwise returns:
    - full holdings array
    - normalized uppercase `symbols`
    - client metadata from parsed input and profile
- **Key expressions or variables used:**  
  - `$input.all().map(item => item.json)`
  - `$('Parse Client Message').first().json`
  - `$('Get Client Profile').first().json`
- **Input and output connections:**  
  - Input from `Get Client Holdings`
  - Output to `IF Holdings Found?`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - Missing symbol values create incomplete price requests
  - No deduplication of symbols, so repeated symbols may be sent multiple times
- **Sub-workflow reference:**  
  - None.

#### 12) IF Holdings Found?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether holdings exist.
- **Configuration choices:**  
  - Condition checks `{{ $json.holdings_found }}` is true.
- **Key expressions or variables used:**  
  - `={{ $json.holdings_found }}`
- **Input and output connections:**  
  - Input from `Prepare Symbols`
  - True branch → `Get Live Prices`
  - False branch → `Build No Holdings Response`
- **Version-specific requirements:**  
  - IF node version `2.2`
- **Edge cases or potential failure types:**  
  - Strict boolean behavior depends on code node returning actual boolean.
- **Sub-workflow reference:**  
  - None.

#### 13) Build No Holdings Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Produces a standardized failure result when the client has no holdings rows.
- **Configuration choices:**  
  - Returns error stage `holdings_lookup`
  - Includes client ID, question, reply, and timestamp
- **Key expressions or variables used:**  
  - `$input.first().json`
- **Input and output connections:**  
  - Input from false branch of `IF Holdings Found?`
  - Output to `Log Result`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - If upstream payload lacks expected fields, some values become null/default.
- **Sub-workflow reference:**  
  - None.

---

## 2.3 Price Fetch and Portfolio Calculation

**Overview:**  
This block sends the client’s symbols to an external price API, normalizes the returned structure, and computes portfolio metrics across all holdings. It also handles partial or total price lookup failures.

**Nodes Involved:**  
- Get Live Prices
- Normalize Price Response
- IF Price API Worked?
- Build API Failed Response
- Merge Portfolio Data

### Node Details

#### 14) Get Live Prices
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls an external live market quote API.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://memic-nse-quotes-api.hf.space/quotes`
  - Timeout: 30000 ms
  - Request body sends:
    - `symbols = {{ $json.symbols }}`
  - `onError` is `continueRegularOutput`, so failures do not hard-stop execution.
- **Key expressions or variables used:**  
  - `={{ $json.symbols }}`
- **Input and output connections:**  
  - Input from true branch of `IF Holdings Found?`
  - Output to `Normalize Price Response`
- **Version-specific requirements:**  
  - HTTP Request node version `4.4`
- **Edge cases or potential failure types:**  
  - External API downtime
  - Timeout
  - Unexpected response schema
  - Invalid symbols
  - Since errors continue, downstream must interpret empty/broken responses
- **Sub-workflow reference:**  
  - None.

#### 15) Normalize Price Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Merges upstream holdings payload with the raw price API response into one normalized object.
- **Configuration choices:**  
  - Reads original holdings package from `Prepare Symbols`
  - Reads first API result
  - Extracts:
    - `prices`
    - `errors`
    - `api_worked`
    - `raw_price_response`
- **Key expressions or variables used:**  
  - `$('Prepare Symbols').first().json`
  - `api.prices || {}`
  - `api.errors || {}`
  - `apiWorked = !!api.prices || Object.keys(api).length > 0`
- **Input and output connections:**  
  - Input from `Get Live Prices`
  - Output to `IF Price API Worked?`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - `api_worked` may become true even for malformed but non-empty responses
  - Partial pricing failures are handled later per symbol
- **Sub-workflow reference:**  
  - None.

#### 16) IF Price API Worked?
- **Type and technical role:** `n8n-nodes-base.if`  
  Determines whether the price response is usable at all.
- **Configuration choices:**  
  - Checks `{{ $json.api_worked }}` is true
- **Key expressions or variables used:**  
  - `={{ $json.api_worked }}`
- **Input and output connections:**  
  - Input from `Normalize Price Response`
  - True branch → `Merge Portfolio Data`
  - False branch → `Build API Failed Response`
- **Version-specific requirements:**  
  - IF node version `2.2`
- **Edge cases or potential failure types:**  
  - Because the upstream check is permissive, some bad payloads may still continue.
- **Sub-workflow reference:**  
  - None.

#### 17) Build API Failed Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Creates a user-facing failure message if live prices could not be retrieved.
- **Configuration choices:**  
  - Sets `error_stage` to `market_api`
  - Returns a generic retry message
- **Key expressions or variables used:**  
  - `$input.first().json`
- **Input and output connections:**  
  - Input from false branch of `IF Price API Worked?`
  - Output to `Log Result`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - If upstream item lacks expected fields, output may be partially null.
- **Sub-workflow reference:**  
  - None.

#### 18) Merge Portfolio Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Performs the core financial calculations by combining holdings and live prices.
- **Configuration choices:**  
  - For each holding:
    - normalizes symbol
    - converts quantity and average buy price to numbers
    - computes invested amount
    - reads `last_price`
    - computes:
      - `current_value`
      - `pnl`
      - `pnl_pct`
      - `price_available`
      - `price_error`
  - Portfolio summary includes:
    - `total_invested_all`
    - `total_invested_priced`
    - `total_current_value`
    - `total_pnl`
    - `total_return_pct`
    - `priced_holdings_count`
    - `missing_price_count`
  - Finds:
    - `best_performer`
    - `weakest_performer`
  - Produces `missing_prices` list
- **Key expressions or variables used:**  
  - `const holdings = data.holdings || []`
  - `const prices = data.prices || {}`
  - `const errors = data.price_errors || {}`
- **Input and output connections:**  
  - Input from true branch of `IF Price API Worked?`
  - Output to `Get Market Context`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - Numeric coercion of bad sheet values becomes `0`
  - Missing `last_price` causes holding exclusion from performance summary
  - `total_return_pct` is based only on priced holdings, not all holdings
  - If no valid priced holdings exist, best/weakest performers become null
- **Sub-workflow reference:**  
  - None.

---

## 2.4 Market Context and AI Answer Generation

**Overview:**  
This block enriches the computed portfolio with optional market context, builds a tightly controlled AI prompt, generates a Gemini answer, and extracts plain text from the model response.

**Nodes Involved:**  
- Get Market Context
- Attach Market Context
- Build AI Prompt
- Generate AI Answer
- Extract AI Answer

### Node Details

#### 19) Get Market Context
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls an external market summary endpoint.
- **Configuration choices:**  
  - Method defaults to GET
  - URL: `https://memic-nse-quotes-api.hf.space/summary`
  - Timeout: 15000 ms
  - `onError` is `continueRegularOutput`
- **Key expressions or variables used:**  
  - None
- **Input and output connections:**  
  - Input from `Merge Portfolio Data`
  - Output to `Attach Market Context`
- **Version-specific requirements:**  
  - HTTP Request node version `4.4`
- **Edge cases or potential failure types:**  
  - API downtime
  - Timeout
  - Empty response
  - Invalid schema
- **Sub-workflow reference:**  
  - None.

#### 20) Attach Market Context
- **Type and technical role:** `n8n-nodes-base.code`  
  Adds optional market-level indicators to the portfolio object.
- **Configuration choices:**  
  - Reads portfolio payload from `Merge Portfolio Data`
  - Maps market fields with safe fallbacks:
    - `nifty_change_pct`
    - `sensex_change_pct`
    - `market_tone`
    - `summary`
- **Key expressions or variables used:**  
  - `$('Merge Portfolio Data').first().json`
  - `$input.first().json`
- **Input and output connections:**  
  - Input from `Get Market Context`
  - Output to `Build AI Prompt`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - Missing API values are handled gracefully with null/default text.
- **Sub-workflow reference:**  
  - None.

#### 21) Build AI Prompt
- **Type and technical role:** `n8n-nodes-base.code`  
  Constructs the final prompt sent to Gemini using all computed facts.
- **Configuration choices:**  
  - Builds formatted text for:
    - holdings details
    - missing price holdings
    - client profile
    - portfolio summary
    - best/weakest performer
    - optional market context
  - Includes strict instruction rules:
    - use only provided numbers
    - no invented facts
    - no extra calculations
    - max 4 short lines
    - fixed answer order
- **Key expressions or variables used:**  
  - Uses `data.holdings`, `data.portfolio_summary`, `data.best_performer`, `data.market_context`
  - Formats numbers with `Number(...).toFixed(2)`
- **Input and output connections:**  
  - Input from `Attach Market Context`
  - Output to `Generate AI Answer`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - Large holdings lists can produce long prompts
  - Quotation/escaping issues may arise if raw text contains unexpected characters
  - The instruction “Never calculate new numbers” is somewhat in tension with asking the model to answer directly from a summary, but most needed figures are already precomputed
- **Sub-workflow reference:**  
  - None.

#### 22) Generate AI Answer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Sends the prompt to Google Gemini and returns the model output.
- **Configuration choices:**  
  - Model: `models/gemini-3.1-flash-lite-preview`
  - Sends one message containing `{{ $json.prompt }}`
  - No built-in tools enabled
  - `onError` is `continueRegularOutput`
- **Key expressions or variables used:**  
  - `={{ $json.prompt }}`
- **Input and output connections:**  
  - Input from `Build AI Prompt`
  - Output to `Extract AI Answer`
- **Version-specific requirements:**  
  - Gemini node version `1.1`
  - Requires Google Gemini/PaLM API credentials.
- **Edge cases or potential failure types:**  
  - Credential issues
  - Model deprecation or access restriction
  - Empty or differently structured response
  - Rate limiting
  - Because errors continue, downstream extraction must detect missing text
- **Sub-workflow reference:**  
  - None.

#### 23) Extract AI Answer
- **Type and technical role:** `n8n-nodes-base.code`  
  Extracts plain text from possible Gemini response formats and flags success/failure.
- **Configuration choices:**  
  - Checks multiple possible response paths:
    - `res.content.parts[0].text`
    - `res.candidates[0].content.parts[0].text`
    - `res.text`
  - Returns `success: false` if no reply found.
- **Key expressions or variables used:**  
  - `$('Build AI Prompt').first().json`
  - `res.content?.parts?.[0]?.text`
  - `res.candidates?.[0]?.content?.parts?.[0]?.text`
- **Input and output connections:**  
  - Input from `Generate AI Answer`
  - Output to `Format Final Output`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - Response format changes from Gemini
  - Empty AI output
  - Null response after continued error
- **Sub-workflow reference:**  
  - None.

---

## 2.5 Final Response, Logging, and Delivery

**Overview:**  
This block formats the response payload, determines whether the AI output is usable, logs all terminal outcomes, optionally creates a Gmail draft for successful replies, and returns a final compact object to the chat caller.

**Nodes Involved:**  
- Format Final Output
- IF AI Reply Exists?
- Build AI Failed Response
- Build Success Result
- Log Result
- Create Gmail Draft
- Return Final Output
- Build Client Not Found Response
- Build No Holdings Response
- Build API Failed Response

### Node Details

#### 24) Format Final Output
- **Type and technical role:** `n8n-nodes-base.set`  
  Standardizes the extracted AI payload into a fixed JSON schema.
- **Configuration choices:**  
  - Raw JSON mode
  - Produces:
    - `success`
    - `client_id`
    - `question`
    - `reply`
    - `source_channel`
    - `received_at`
    - `response_type = portfolio_answer`
    - `timestamp = $now.toISO()`
- **Key expressions or variables used:**  
  - `{{ $json.success }}`
  - `{{$json.client_id}}`
  - `$now.toISO()`
- **Input and output connections:**  
  - Input from `Extract AI Answer`
  - Output to `IF AI Reply Exists?`
- **Version-specific requirements:**  
  - Set node version `3.4`
- **Edge cases or potential failure types:**  
  - Raw JSON interpolation may break if `reply` contains unescaped quotes/newlines.
  - This is one of the more fragile nodes in the workflow.
- **Sub-workflow reference:**  
  - None.

#### 25) IF AI Reply Exists?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches on AI success status.
- **Configuration choices:**  
  - Checks `{{ $json.success }}` is true
- **Key expressions or variables used:**  
  - `={{ $json.success }}`
- **Input and output connections:**  
  - Input from `Format Final Output`
  - True branch → `Build Success Result`
  - False branch → `Build AI Failed Response`
- **Version-specific requirements:**  
  - IF node version `2.2`
- **Edge cases or potential failure types:**  
  - If `success` is not a strict boolean, branch behavior may surprise.
- **Sub-workflow reference:**  
  - None.

#### 26) Build AI Failed Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Generates a terminal failure payload when AI output is empty.
- **Configuration choices:**  
  - Sets `error_stage = ai_generation`
  - Generic retry message
- **Key expressions or variables used:**  
  - `$input.first().json`
- **Input and output connections:**  
  - Input from false branch of `IF AI Reply Exists?`
  - Output to `Log Result`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - Upstream malformed payload may leave some fields null.
- **Sub-workflow reference:**  
  - None.

#### 27) Build Success Result
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts successful AI output into the canonical logged result structure.
- **Configuration choices:**  
  - Sets:
    - `status = success`
    - `action_taken = Generated portfolio reply`
- **Key expressions or variables used:**  
  - `$input.first().json`
- **Input and output connections:**  
  - Input from true branch of `IF AI Reply Exists?`
  - Output to `Log Result`
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - If reply text is malformed, it still logs as success if `success` was true.
- **Sub-workflow reference:**  
  - None.

#### 28) Log Result
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends final success/failure events to the interaction log sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet: `Portfolio_AI_System`
  - Sheet: `interaction_logs`
  - Explicit mappings for:
    - reply
    - status
    - question
    - client_id
    - timestamp
    - error_stage
    - action_taken
    - source_channel
- **Key expressions or variables used:**  
  - Expressions map each output field explicitly, e.g. `={{ $json.reply }}`
- **Input and output connections:**  
  - Inputs from:
    - `Build Success Result`
    - `Build AI Failed Response`
    - `Build API Failed Response`
    - `Build No Holdings Response`
    - `Build Client Not Found Response`
  - Output to `Return Final Output`
- **Version-specific requirements:**  
  - Google Sheets node version `4.7`
- **Edge cases or potential failure types:**  
  - Logging failure prevents final downstream return on those branches
  - Sheet schema drift can break appends
- **Sub-workflow reference:**  
  - None.

#### 29) Create Gmail Draft
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Creates a Gmail draft containing the generated reply.
- **Configuration choices:**  
  - Resource: `draft`
  - Subject: `Portfolio reply for <client_id>`
  - Message body: `{{ $json.reply }}`
  - No recipient is specified because it is a draft, not a send action
- **Key expressions or variables used:**  
  - `={{ $json.reply }}`
  - `={{ 'Portfolio reply for ' + ($json.client_id || 'client') }}`
- **Input and output connections:**  
  - No inbound connection in the provided workflow
  - Output to `Return Final Output`
- **Version-specific requirements:**  
  - Gmail node version `2.1`
  - Requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - This node is currently orphaned and will never run.
  - If connected later, Gmail auth and mailbox permissions must be valid.
- **Sub-workflow reference:**  
  - None.

#### 30) Return Final Output
- **Type and technical role:** `n8n-nodes-base.code`  
  Produces the final compact response returned to the chat trigger.
- **Configuration choices:**  
  - Returns:
    - `final_status`
    - `client_id`
    - `question`
    - `reply`
    - `action_taken`
- **Key expressions or variables used:**  
  - `$json.status`, `$json.client_id`, `$json.question`, `$json.reply`, `$json.action_taken`
- **Input and output connections:**  
  - Inputs from:
    - `Log Result`
    - `Create Gmail Draft`
  - Final terminal node for successful and most failure branches
- **Version-specific requirements:**  
  - Code node version `2`
- **Edge cases or potential failure types:**  
  - If fed from `Create Gmail Draft`, expected fields like `status` may not exist unless that branch is pre-shaped first.
- **Sub-workflow reference:**  
  - None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat entry point |  | Parse Client Message | ##  Input Handling & Validation<br>This section handles incoming chat messages and ensures they follow the correct format.<br><br>It extracts the client ID and question and checks for common issues like empty input or missing separators. If validation fails, the workflow immediately stops, logs the issue and returns a clean error response.<br><br>This keeps the rest of the workflow safe from bad data. |
| Parse Client Message | `n8n-nodes-base.code` | Parse and validate inbound message | When chat message received | IF Valid Input? | ##  Input Handling & Validation<br>This section handles incoming chat messages and ensures they follow the correct format.<br><br>It extracts the client ID and question and checks for common issues like empty input or missing separators. If validation fails, the workflow immediately stops, logs the issue and returns a clean error response.<br><br>This keeps the rest of the workflow safe from bad data. |
| IF Valid Input? | `n8n-nodes-base.if` | Branch on input validity | Parse Client Message | Get Client Profile; Build Invalid Input Response | ##  Input Handling & Validation<br>This section handles incoming chat messages and ensures they follow the correct format.<br><br>It extracts the client ID and question and checks for common issues like empty input or missing separators. If validation fails, the workflow immediately stops, logs the issue and returns a clean error response.<br><br>This keeps the rest of the workflow safe from bad data. |
| Build Invalid Input Response | `n8n-nodes-base.code` | Build invalid-input failure payload | IF Valid Input? | Log Invalid Input | ##  Input Handling & Validation<br>This section handles incoming chat messages and ensures they follow the correct format.<br><br>It extracts the client ID and question and checks for common issues like empty input or missing separators. If validation fails, the workflow immediately stops, logs the issue and returns a clean error response.<br><br>This keeps the rest of the workflow safe from bad data. |
| Log Invalid Input | `n8n-nodes-base.googleSheets` | Log invalid request to Google Sheets | Build Invalid Input Response | Return Invalid Response | ##  Input Handling & Validation<br>This section handles incoming chat messages and ensures they follow the correct format.<br><br>It extracts the client ID and question and checks for common issues like empty input or missing separators. If validation fails, the workflow immediately stops, logs the issue and returns a clean error response.<br><br>This keeps the rest of the workflow safe from bad data. |
| Return Invalid Response | `n8n-nodes-base.code` | Return short failure payload for invalid input | Log Invalid Input |  | ##  Input Handling & Validation<br>This section handles incoming chat messages and ensures they follow the correct format.<br><br>It extracts the client ID and question and checks for common issues like empty input or missing separators. If validation fails, the workflow immediately stops, logs the issue and returns a clean error response.<br><br>This keeps the rest of the workflow safe from bad data. |
| Get Client Profile | `n8n-nodes-base.googleSheets` | Retrieve client row from clients sheet | IF Valid Input? | IF Client Found? | ## Client & Portfolio Lookup<br>Here, the workflow fetches client profile and holdings from Google Sheets using the client ID.<br><br>It also prepares stock symbols required for fetching live prices. If the client doesn’t exist or no holdings are found, the workflow stops early and returns a clear message.<br><br>This ensures only valid and complete data moves forward. |
| IF Client Found? | `n8n-nodes-base.if` | Branch on client existence | Get Client Profile | Get Client Holdings; Build Client Not Found Response | ## Client & Portfolio Lookup<br>Here, the workflow fetches client profile and holdings from Google Sheets using the client ID.<br><br>It also prepares stock symbols required for fetching live prices. If the client doesn’t exist or no holdings are found, the workflow stops early and returns a clear message.<br><br>This ensures only valid and complete data moves forward. |
| Build Client Not Found Response | `n8n-nodes-base.code` | Build failure when client is missing | IF Client Found? | Log Result |  |
| Get Client Holdings | `n8n-nodes-base.googleSheets` | Retrieve holdings rows for client | IF Client Found? | Prepare Symbols | ## Client & Portfolio Lookup<br>Here, the workflow fetches client profile and holdings from Google Sheets using the client ID.<br><br>It also prepares stock symbols required for fetching live prices. If the client doesn’t exist or no holdings are found, the workflow stops early and returns a clear message.<br><br>This ensures only valid and complete data moves forward. |
| Prepare Symbols | `n8n-nodes-base.code` | Normalize holdings and extract symbols | Get Client Holdings | IF Holdings Found? | ## Client & Portfolio Lookup<br>Here, the workflow fetches client profile and holdings from Google Sheets using the client ID.<br><br>It also prepares stock symbols required for fetching live prices. If the client doesn’t exist or no holdings are found, the workflow stops early and returns a clear message.<br><br>This ensures only valid and complete data moves forward. |
| IF Holdings Found? | `n8n-nodes-base.if` | Branch on holdings existence | Prepare Symbols | Get Live Prices; Build No Holdings Response | ## Client & Portfolio Lookup<br>Here, the workflow fetches client profile and holdings from Google Sheets using the client ID.<br><br>It also prepares stock symbols required for fetching live prices. If the client doesn’t exist or no holdings are found, the workflow stops early and returns a clear message.<br><br>This ensures only valid and complete data moves forward. |
| Build No Holdings Response | `n8n-nodes-base.code` | Build failure when client has no holdings | IF Holdings Found? | Log Result |  |
| Get Live Prices | `n8n-nodes-base.httpRequest` | Fetch live quote data | IF Holdings Found? | Normalize Price Response | ##  Price Fetch & Portfolio Calculation<br>This part calls the live market API to fetch prices for all holdings.<br><br>It then merges this data with the portfolio and calculates key metrics like invested value, current value, profit/loss and returns. It also identifies the best and weakest performers and tracks any missing price data.<br><br>This step builds a complete and accurate portfolio snapshot. |
| Normalize Price Response | `n8n-nodes-base.code` | Normalize raw quote API response | Get Live Prices | IF Price API Worked? | ##  Price Fetch & Portfolio Calculation<br>This part calls the live market API to fetch prices for all holdings.<br><br>It then merges this data with the portfolio and calculates key metrics like invested value, current value, profit/loss and returns. It also identifies the best and weakest performers and tracks any missing price data.<br><br>This step builds a complete and accurate portfolio snapshot. |
| IF Price API Worked? | `n8n-nodes-base.if` | Branch on quote API usability | Normalize Price Response | Merge Portfolio Data; Build API Failed Response | ##  Price Fetch & Portfolio Calculation<br>This part calls the live market API to fetch prices for all holdings.<br><br>It then merges this data with the portfolio and calculates key metrics like invested value, current value, profit/loss and returns. It also identifies the best and weakest performers and tracks any missing price data.<br><br>This step builds a complete and accurate portfolio snapshot. |
| Build API Failed Response | `n8n-nodes-base.code` | Build failure when live price API is unavailable | IF Price API Worked? | Log Result |  |
| Merge Portfolio Data | `n8n-nodes-base.code` | Compute holding and portfolio metrics | IF Price API Worked? | Get Market Context | ##  Price Fetch & Portfolio Calculation<br>This part calls the live market API to fetch prices for all holdings.<br><br>It then merges this data with the portfolio and calculates key metrics like invested value, current value, profit/loss and returns. It also identifies the best and weakest performers and tracks any missing price data.<br><br>This step builds a complete and accurate portfolio snapshot. |
| Get Market Context | `n8n-nodes-base.httpRequest` | Fetch optional market summary | Merge Portfolio Data | Attach Market Context | ## AI Answer Generation<br>A structured prompt is created using portfolio data, client profile and optional market context.<br><br>The AI generates a short, clear response based strictly on the provided data. Rules are applied to avoid incorrect calculations or assumptions.<br><br>The final answer is extracted and cleaned before sending forward. |
| Attach Market Context | `n8n-nodes-base.code` | Merge portfolio data with market summary | Get Market Context | Build AI Prompt | ## AI Answer Generation<br>A structured prompt is created using portfolio data, client profile and optional market context.<br><br>The AI generates a short, clear response based strictly on the provided data. Rules are applied to avoid incorrect calculations or assumptions.<br><br>The final answer is extracted and cleaned before sending forward. |
| Build AI Prompt | `n8n-nodes-base.code` | Construct constrained prompt for AI | Attach Market Context | Generate AI Answer | ## AI Answer Generation<br>A structured prompt is created using portfolio data, client profile and optional market context.<br><br>The AI generates a short, clear response based strictly on the provided data. Rules are applied to avoid incorrect calculations or assumptions.<br><br>The final answer is extracted and cleaned before sending forward. |
| Generate AI Answer | `@n8n/n8n-nodes-langchain.googleGemini` | Generate natural-language answer with Gemini | Build AI Prompt | Extract AI Answer | ## AI Answer Generation<br>A structured prompt is created using portfolio data, client profile and optional market context.<br><br>The AI generates a short, clear response based strictly on the provided data. Rules are applied to avoid incorrect calculations or assumptions.<br><br>The final answer is extracted and cleaned before sending forward. |
| Extract AI Answer | `n8n-nodes-base.code` | Extract plain text from Gemini response | Generate AI Answer | Format Final Output | ## AI Answer Generation<br>A structured prompt is created using portfolio data, client profile and optional market context.<br><br>The AI generates a short, clear response based strictly on the provided data. Rules are applied to avoid incorrect calculations or assumptions.<br><br>The final answer is extracted and cleaned before sending forward. |
| Format Final Output | `n8n-nodes-base.set` | Standardize AI output payload | Extract AI Answer | IF AI Reply Exists? | ##  Final Response & Logging<br>The workflow formats the final output with the reply, client details and status.<br><br>It can also log interactions or be extended to send responses to other systems like CRM, email or Slack.<br><br>This acts as the final delivery layer of the workflow. |
| IF AI Reply Exists? | `n8n-nodes-base.if` | Branch on AI extraction success | Format Final Output | Build Success Result; Build AI Failed Response | ##  Final Response & Logging<br>The workflow formats the final output with the reply, client details and status.<br><br>It can also log interactions or be extended to send responses to other systems like CRM, email or Slack.<br><br>This acts as the final delivery layer of the workflow. |
| Build AI Failed Response | `n8n-nodes-base.code` | Build failure when AI returns no text | IF AI Reply Exists? | Log Result | ##  Final Response & Logging<br>The workflow formats the final output with the reply, client details and status.<br><br>It can also log interactions or be extended to send responses to other systems like CRM, email or Slack.<br><br>This acts as the final delivery layer of the workflow. |
| Build Success Result | `n8n-nodes-base.code` | Build final success log payload | IF AI Reply Exists? | Log Result | ##  Final Response & Logging<br>The workflow formats the final output with the reply, client details and status.<br><br>It can also log interactions or be extended to send responses to other systems like CRM, email or Slack.<br><br>This acts as the final delivery layer of the workflow. |
| Log Result | `n8n-nodes-base.googleSheets` | Log terminal workflow outcome | Build Success Result; Build AI Failed Response; Build API Failed Response; Build No Holdings Response; Build Client Not Found Response | Return Final Output | ##  Final Response & Logging<br>The workflow formats the final output with the reply, client details and status.<br><br>It can also log interactions or be extended to send responses to other systems like CRM, email or Slack.<br><br>This acts as the final delivery layer of the workflow. |
| Create Gmail Draft | `n8n-nodes-base.gmail` | Prepare draft email with generated response |  | Return Final Output | ##  Final Response & Logging<br>The workflow formats the final output with the reply, client details and status.<br><br>It can also log interactions or be extended to send responses to other systems like CRM, email or Slack.<br><br>This acts as the final delivery layer of the workflow. |
| Return Final Output | `n8n-nodes-base.code` | Return compact final response to caller | Log Result; Create Gmail Draft |  | ##  Final Response & Logging<br>The workflow formats the final output with the reply, client details and status.<br><br>It can also log interactions or be extended to send responses to other systems like CRM, email or Slack.<br><br>This acts as the final delivery layer of the workflow. |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ## Client Portfolio Assistant<br><br>### How it works<br>This workflow takes a client message (e.g., "C001: question"), validates the format and identifies the client. It fetches client profile and holdings from Google Sheets, pulls live market prices and calculates portfolio metrics like total value, P&L and returns. Optional market context is added for better insights. An AI model then generates a short, accurate and client-friendly response based only on this data. All interactions are logged for tracking and reliability.<br><br>### Setup steps<br>1. Create Google Sheets with clients, holdings and logs.<br>2. Connect Google Sheets credentials in n8n.<br>3. Add live price API endpoint.<br>4. Add market summary API (optional).<br>5. Configure AI node (Gemini or HF).<br>6. Test using: `C001: your question` |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ##  Input Handling & Validation<br>This section handles incoming chat messages and ensures they follow the correct format.<br><br>It extracts the client ID and question and checks for common issues like empty input or missing separators. If validation fails, the workflow immediately stops, logs the issue and returns a clean error response.<br><br>This keeps the rest of the workflow safe from bad data. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ## Client & Portfolio Lookup<br>Here, the workflow fetches client profile and holdings from Google Sheets using the client ID.<br><br>It also prepares stock symbols required for fetching live prices. If the client doesn’t exist or no holdings are found, the workflow stops early and returns a clear message.<br><br>This ensures only valid and complete data moves forward. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ##  Price Fetch & Portfolio Calculation<br>This part calls the live market API to fetch prices for all holdings.<br><br>It then merges this data with the portfolio and calculates key metrics like invested value, current value, profit/loss and returns. It also identifies the best and weakest performers and tracks any missing price data.<br><br>This step builds a complete and accurate portfolio snapshot. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ## AI Answer Generation<br>A structured prompt is created using portfolio data, client profile and optional market context.<br><br>The AI generates a short, clear response based strictly on the provided data. Rules are applied to avoid incorrect calculations or assumptions.<br><br>The final answer is extracted and cleaned before sending forward. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Canvas documentation note |  |  | ##  Final Response & Logging<br>The workflow formats the final output with the reply, client details and status.<br><br>It can also log interactions or be extended to send responses to other systems like CRM, email or Slack.<br><br>This acts as the final delivery layer of the workflow. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the required Google Sheets workbook**
   - Create a spreadsheet named similar to `Portfolio_AI_System`.
   - Add at least these sheets:
     1. `clients`
     2. `holdings`
     3. `interaction_logs`

2. **Prepare the `clients` sheet**
   - Include columns such as:
     - `client_id`
     - `client_name`
     - `risk_profile`
     - `goal`
     - `tone`
   - Ensure `client_id` values are unique, e.g. `C001`.

3. **Prepare the `holdings` sheet**
   - Include columns such as:
     - `row_number`
     - `client_id`
     - `asset_name`
     - `symbol`
     - `asset_type`
     - `quantity`
     - `avg_buy_price`
   - Ensure each client can have multiple rows.

4. **Prepare the `interaction_logs` sheet**
   - Include columns:
     - `timestamp`
     - `client_id`
     - `question`
     - `reply`
     - `status`
     - `action_taken`
     - `error_stage`
     - `source_channel`

5. **Create Google Sheets credentials in n8n**
   - Add a `Google Sheets OAuth2 API` credential.
   - Ensure the authenticated account can read and write the workbook.

6. **Create Google Gemini credentials**
   - Add a `Google Gemini/PaLM API` credential in n8n.
   - Verify access to the selected model or update the model to one available in your environment.

7. **Optionally create Gmail credentials**
   - Add `Gmail OAuth2` credentials.
   - This is only needed if you plan to connect the Gmail draft step.

8. **Create the trigger node**
   - Add a `When chat message received` node of type `Chat Trigger`.
   - Set response mode to `Last Node`.

9. **Create `Parse Client Message`**
   - Add a `Code` node after the trigger.
   - Set mode to `Run Once for Each Item`.
   - Paste the parsing logic to:
     - read `chatInput`, `message`, or `text`
     - validate non-empty input
     - validate presence of `:`
     - split into `client_id` and `question`
     - return `valid`, metadata, and timestamps

10. **Create `IF Valid Input?`**
    - Add an `If` node.
    - Condition:
      - left value: `{{ $json.valid }}`
      - operation: `is true`

11. **Create invalid-input handling**
    - Add `Build Invalid Input Response` as a `Code` node on the false branch.
    - Build a JSON object with:
      - `status = failed`
      - `error_stage`
      - `action_taken`
      - `reply`
      - `client_id`
      - `question`
      - `source_channel`
      - `timestamp`

12. **Create `Log Invalid Input`**
    - Add a `Google Sheets` node after invalid response.
    - Operation: `Append`
    - Sheet: `interaction_logs`
    - Map the eight log columns.

13. **Create `Return Invalid Response`**
    - Add a `Code` node after `Log Invalid Input`.
    - Return only:
      - `reply`
      - `status`
      - `error_stage`

14. **Create `Get Client Profile`**
    - On the true branch of `IF Valid Input?`, add a `Google Sheets` node.
    - Operation: read rows/filter rows.
    - Sheet: `clients`
    - Filter:
      - `client_id` equals `{{ $json.client_id }}`

15. **Create `IF Client Found?`**
    - Add an `If` node after `Get Client Profile`.
    - Condition:
      - `{{ !!$json.client_id }}`
      - is true

16. **Create `Build Client Not Found Response`**
    - Add a `Code` node on the false branch.
    - Reference `Parse Client Message` to recover the original `client_id` and `question`.
    - Return a failure payload with:
      - `status = failed`
      - `error_stage = client_lookup`
      - `reply = Client ID ... was not found...`

17. **Create `Get Client Holdings`**
    - On the true branch of `IF Client Found?`, add a `Google Sheets` node.
    - Sheet: `holdings`
    - Filter:
      - `client_id` equals `{{ $('Parse Client Message').first().json.client_id }}`

18. **Create `Prepare Symbols`**
    - Add a `Code` node after `Get Client Holdings`.
    - Logic should:
      - collect all rows
      - filter empty objects
      - return `holdings_found = false` if none exist
      - else produce:
        - `holdings`
        - `symbols`
        - `client_id`
        - `question`
        - `client_name`
        - `risk_profile`
        - `goal`
        - `tone`
        - `source_channel`
        - `received_at`

19. **Create `IF Holdings Found?`**
    - Add an `If` node.
    - Condition:
      - `{{ $json.holdings_found }}`
      - is true

20. **Create `Build No Holdings Response`**
    - Add a `Code` node on the false branch.
    - Return:
      - `status = failed`
      - `error_stage = holdings_lookup`
      - reply indicating no holdings found

21. **Create `Get Live Prices`**
    - On the true branch of `IF Holdings Found?`, add an `HTTP Request` node.
    - Method: `POST`
    - URL: `https://memic-nse-quotes-api.hf.space/quotes`
    - Timeout: `30000`
    - Body parameter:
      - `symbols = {{ $json.symbols }}`
    - Set node error handling to continue output on error.

22. **Create `Normalize Price Response`**
    - Add a `Code` node after `Get Live Prices`.
    - Merge:
      - original prepared holdings package
      - API response
    - Return:
      - `api_worked`
      - `prices`
      - `price_errors`
      - `raw_price_response`

23. **Create `IF Price API Worked?`**
    - Add an `If` node.
    - Condition:
      - `{{ $json.api_worked }}`
      - is true

24. **Create `Build API Failed Response`**
    - Add a `Code` node on the false branch.
    - Return:
      - `status = failed`
      - `error_stage = market_api`
      - a retry-style user reply

25. **Create `Merge Portfolio Data`**
    - On the true branch, add a `Code` node.
    - For each holding:
      - normalize symbol
      - convert `quantity` and `avg_buy_price` to numbers
      - pull `last_price`
      - compute invested, current value, pnl, pnl_pct
    - Also compute:
      - portfolio totals
      - best performer
      - weakest performer
      - missing prices list

26. **Create `Get Market Context`**
    - Add an `HTTP Request` node after `Merge Portfolio Data`.
    - URL: `https://memic-nse-quotes-api.hf.space/summary`
    - Timeout: `15000`
    - Enable continue on error.

27. **Create `Attach Market Context`**
    - Add a `Code` node after `Get Market Context`.
    - Merge market data into the portfolio payload with safe defaults.

28. **Create `Build AI Prompt`**
    - Add a `Code` node.
    - Build a prompt containing:
      - client profile
      - original question
      - portfolio totals
      - best/weakest performer
      - holdings details
      - missing prices
      - optional market context
    - Include strict instruction lines:
      - only use provided facts
      - do not invent reasons
      - no buy/sell advice unless asked
      - maximum 4 short lines

29. **Create `Generate AI Answer`**
    - Add a `Google Gemini` node.
    - Select model:
      - `models/gemini-3.1-flash-lite-preview` if available
    - Add one message with content:
      - `{{ $json.prompt }}`
    - Leave tools disabled unless specifically needed.
    - Set continue on error.

30. **Create `Extract AI Answer`**
    - Add a `Code` node after Gemini.
    - Extract the text from possible locations:
      - `content.parts[0].text`
      - `candidates[0].content.parts[0].text`
      - `text`
    - Return:
      - `success`
      - `client_id`
      - `question`
      - `source_channel`
      - `received_at`
      - `reply`

31. **Create `Format Final Output`**
    - Add a `Set` node in raw JSON mode.
    - Build a normalized object with:
      - `success`
      - `client_id`
      - `question`
      - `reply`
      - `source_channel`
      - `received_at`
      - `response_type = portfolio_answer`
      - `timestamp = $now.toISO()`
   - Recommended improvement: prefer field-by-field Set mapping instead of raw JSON string interpolation to avoid escaping issues.

32. **Create `IF AI Reply Exists?`**
    - Add an `If` node.
    - Condition:
      - `{{ $json.success }}`
      - is true

33. **Create `Build AI Failed Response`**
    - Add a `Code` node on the false branch.
    - Return:
      - `status = failed`
      - `error_stage = ai_generation`
      - generic retry reply

34. **Create `Build Success Result`**
    - Add a `Code` node on the true branch.
    - Return:
      - `status = success`
      - `action_taken = Generated portfolio reply`
      - plus `client_id`, `question`, `reply`, `source_channel`, `timestamp`

35. **Create `Log Result`**
    - Add a `Google Sheets` append node.
    - Sheet: `interaction_logs`
    - Map:
      - `reply`
      - `status`
      - `question`
      - `client_id`
      - `timestamp`
      - `error_stage`
      - `action_taken`
      - `source_channel`
    - Connect all terminal result builders to this node:
      - `Build Success Result`
      - `Build AI Failed Response`
      - `Build API Failed Response`
      - `Build No Holdings Response`
      - `Build Client Not Found Response`

36. **Create `Return Final Output`**
    - Add a `Code` node after `Log Result`.
    - Return:
      - `final_status`
      - `client_id`
      - `question`
      - `reply`
      - `action_taken`

37. **Optionally create `Create Gmail Draft`**
    - Add a `Gmail` node.
    - Resource: `Draft`
    - Subject:
      - `Portfolio reply for {{ $json.client_id || 'client' }}`
    - Message:
      - `{{ $json.reply }}`
    - If you want it to run, connect it after `Build Success Result` or after `Log Result` with proper field mapping.
    - In the provided workflow, this node is not connected inbound, so it never executes.

38. **Connect the workflow exactly**
    - `When chat message received` → `Parse Client Message`
    - `Parse Client Message` → `IF Valid Input?`
    - False branch:
      - `Build Invalid Input Response` → `Log Invalid Input` → `Return Invalid Response`
    - True branch:
      - `Get Client Profile` → `IF Client Found?`
    - Client not found branch:
      - `Build Client Not Found Response` → `Log Result`
    - Client found branch:
      - `Get Client Holdings` → `Prepare Symbols` → `IF Holdings Found?`
    - No holdings branch:
      - `Build No Holdings Response` → `Log Result`
    - Holdings found branch:
      - `Get Live Prices` → `Normalize Price Response` → `IF Price API Worked?`
    - API failed branch:
      - `Build API Failed Response` → `Log Result`
    - API success branch:
      - `Merge Portfolio Data` → `Get Market Context` → `Attach Market Context` → `Build AI Prompt` → `Generate AI Answer` → `Extract AI Answer` → `Format Final Output` → `IF AI Reply Exists?`
    - AI failure branch:
      - `Build AI Failed Response` → `Log Result`
    - AI success branch:
      - `Build Success Result` → `Log Result`
    - Final:
      - `Log Result` → `Return Final Output`

39. **Add the documentation sticky notes if desired**
    - Add one high-level note describing the workflow purpose and setup.
    - Add block notes for:
      - input validation
      - client/portfolio lookup
      - price calculation
      - AI answer generation
      - final logging/delivery

40. **Test with representative inputs**
    - Valid:
      - `C001: What is my portfolio performance today?`
    - Invalid:
      - empty message
      - `What is my portfolio performance?`
      - `C001:`
      - `: What is my portfolio performance?`
    - Missing client:
      - `C999: How is my portfolio doing?`
    - Client with no holdings

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Client Portfolio Assistant: This workflow takes a client message (e.g., `C001: question`), validates the format and identifies the client. It fetches client profile and holdings from Google Sheets, pulls live market prices and calculates portfolio metrics like total value, P&L and returns. Optional market context is added for better insights. An AI model then generates a short, accurate and client-friendly response based only on this data. All interactions are logged for tracking and reliability. | Workflow-level description |
| Setup steps noted in the canvas: 1. Create Google Sheets with clients, holdings and logs. 2. Connect Google Sheets credentials in n8n. 3. Add live price API endpoint. 4. Add market summary API (optional). 5. Configure AI node (Gemini or HF). 6. Test using: `C001: your question` | Workflow-level setup note |
| Live quotes API endpoint used for pricing | `https://memic-nse-quotes-api.hf.space/quotes` |
| Market summary API endpoint used for context | `https://memic-nse-quotes-api.hf.space/summary` |
| Google Sheets document used by the workflow | `https://docs.google.com/spreadsheets/d/1Utb59F9XfxP6j--TdE1HOApUk8Gv8CTIS2_BK3voTHM/edit?usp=drivesdk` |

### Additional implementation observations
- The workflow has a single actual entry point: `When chat message received`.
- There are no sub-workflows or Execute Workflow nodes.
- `Create Gmail Draft` is currently disconnected from upstream execution and therefore inactive in practice.
- The `Format Final Output` Set node uses raw JSON string interpolation; this should be reviewed if AI replies may contain quotes or multiline text.
- Logging is split into two paths:
  - invalid input uses `Log Invalid Input`
  - all other terminal outcomes use `Log Result`
- The market context step is optional by design because its HTTP node continues on error and defaults are added later.