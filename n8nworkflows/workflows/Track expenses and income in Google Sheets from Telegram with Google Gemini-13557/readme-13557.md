Track expenses and income in Google Sheets from Telegram with Google Gemini

https://n8nworkflows.xyz/workflows/track-expenses-and-income-in-google-sheets-from-telegram-with-google-gemini-13557


# Track expenses and income in Google Sheets from Telegram with Google Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Track expenses and income in Google Sheets from Telegram with Google Gemini  
**Workflow name (in JSON):** `[FOR TEMPLATE] AI Expense Tracker Agent`  
**Purpose:** A Telegram bot that lets users record expenses and income in Google Sheets using natural language. Google Gemini powers an AI Agent that parses messages, calls Google Sheets “tools” to append/get/delete rows, optionally uses calculator/think tools, and replies back to Telegram with formatted output.

### 1.1 Input Reception (Telegram)
Receives incoming Telegram messages from a bot and immediately signals “typing…” to improve UX.

### 1.2 AI Understanding & Orchestration (Gemini + Agent + Memory)
Gemini Chat Model + an n8n LangChain Agent interpret the user message, decide which tool(s) to call (add/get/delete expense, add/get income, calculator, think), and produce a final response message.

### 1.3 Data Operations (Google Sheets Tools)
Writes/reads/deletes rows in Google Sheets:
- Monthly expense sheet auto-selected by current month name (e.g., “February 2026”)
- Income stored in a static sheet “Income Log”

### 1.4 Output Formatting & Delivery (Telegram HTML)
Converts the agent’s markdown-ish response into Telegram-safe HTML, chunks long messages, and sends them back. If the AI agent path errors, sends a friendly error message.

---

## 2. Block-by-Block Analysis

### Block A — Telegram Intake & UX Feedback
**Overview:** Triggers on new Telegram messages and sends a “typing…” chat action while the AI processes.  
**Nodes involved:** `Telegram Trigger`, `Typing...`

#### Node: Telegram Trigger
- **Type / role:** `n8n-nodes-base.telegramTrigger` — entry point; listens for Telegram updates.
- **Configuration (interpreted):**
  - Updates: `message` (only standard message events).
- **Key variables/expressions:** None; outputs Telegram update payload under `$json.message...`.
- **Connections:**
  - **Output →** `Typing...` (main)
  - **Output →** `AI Agent - Expense Tracker` (main)
- **Edge cases / failures:**
  - Telegram credential/token invalid → trigger won’t receive updates.
  - Bot not started by user / privacy settings in groups can prevent messages arriving.
  - Webhook issues (network/DNS/Telegram webhook not set) can block triggers.
- **Version notes:** Node `typeVersion 1.2` (behavior may differ slightly from older Telegram trigger versions regarding update fields).

#### Node: Typing...
- **Type / role:** `n8n-nodes-base.telegram` — sends “typing…” action to chat.
- **Configuration (interpreted):**
  - Operation: `sendChatAction`
  - Chat ID: `={{ $json.message.chat.id }}`
- **Connections:**
  - **Input ←** `Telegram Trigger`
  - No downstream nodes (UX-only side action).
- **Edge cases / failures:**
  - Chat ID missing for non-message updates (not applicable here since updates are `message`).
  - Telegram rate limits if heavy usage.
- **Version notes:** Node `typeVersion 1.2`.

---

### Block B — AI Agent Orchestration (Gemini + Memory + Tooling)
**Overview:** Uses Gemini as the LLM, keeps short conversation memory keyed per chat, and exposes tools (Google Sheets + calculator/think) for the agent to call.  
**Nodes involved:** `Google Gemini Chat Model`, `Simple Memory`, `AI Agent - Expense Tracker`, and all tool nodes (`add_expense`, `get_expense`, `delete_expense`, `add_income`, `get_income`, `Calculator`, `Think`)

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — LLM backend for the agent.
- **Configuration (interpreted):**
  - Temperature: `0.6` (moderately creative; can slightly increase parsing variability).
- **Connections:**
  - **Output (ai_languageModel) →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - Missing/invalid Gemini API key credential.
  - Model/provider throttling, quota exceeded, transient 5xx.
  - Higher temperature can cause inconsistent categorization/field extraction.
- **Version notes:** `typeVersion 1`.

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — conversational memory store (buffer window) per Telegram chat.
- **Configuration (interpreted):**
  - Session ID: custom key = `={{ $json.message.chat.id }}`
  - This makes each Telegram chat a separate conversation memory.
- **Connections:**
  - **Output (ai_memory) →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - If chat.id missing (unlikely with message updates), memory session key fails.
  - Memory window size defaults apply (not explicitly configured); long histories may be truncated.
- **Version notes:** `typeVersion 1.3`.

#### Node: AI Agent - Expense Tracker
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — central orchestration; interprets user text, calls tools, composes final reply.
- **Configuration (interpreted):**
  - Input text: `={{ $json.message.text }}`
  - Prompt: `define` with a long **system message** defining:
    - Available tools and when to use them
    - Categories/payment method mapping rules
    - Amount parsing rules (e.g., 25k → 25000; 1.5jt → 1500000)
    - Workflows for expense/income/view/delete
    - Response formats and supported commands
    - Injected current time: `{{ $now.format('cccc DD HH:mm') }}`
  - Error handling: `onError: continueErrorOutput` (routes errors to second output).
  - Retry: `retryOnFail: true`, `waitBetweenTries: 5000ms`
- **Connections:**
  - **Main input ←** `Telegram Trigger`
  - **ai_languageModel ←** `Google Gemini Chat Model`
  - **ai_memory ←** `Simple Memory`
  - **ai_tool connections ←** all tool nodes: `add_expense`, `get_expense`, `delete_expense`, `add_income`, `get_income`, `Calculator`, `Think`
  - **Output 1 (success) →** `Markdown to HTML Formatter`
  - **Output 2 (error) →** `Send Error Message`
- **Edge cases / failures:**
  - If Telegram message is not text (photo/voice), `$json.message.text` may be undefined → agent gets empty input (should be handled explicitly if desired).
  - Tool-call schema mismatch: `$fromAI(...)` fields not provided by the model can produce empty strings or invalid types (notably `Number_of_Rows_to_Delete` expects number).
  - The system rules say “ALWAYS call tool first” even for simple help; this can cause unnecessary tool calls unless the LLM chooses otherwise.
  - Delete limitation: only last N rows can be deleted; agent is instructed accordingly, but user expectations may differ.
- **Version notes:** `typeVersion 1.7` (LangChain agent node behavior can change across versions regarding tool invocation and structured outputs).

---

### Block C — Google Sheets Tools (Expenses & Income)
**Overview:** Implements the agent’s “actions” as Google Sheets Tool nodes: append rows for expenses/income, read sheet data, and delete last N rows.  
**Nodes involved:** `add_expense`, `get_expense`, `delete_expense`, `add_income`, `get_income`

Common traits:
- All use `documentId = YOUR_GOOGLE_SHEET_ID_HERE` (must be replaced).
- Expense sheet name is dynamic: `={{ $now.setLocale('en').toFormat('MMMM yyyy') }}` (e.g., “February 2026”).
- Income uses a static sheet: `"Income Log"`.

#### Node: add_expense
- **Type / role:** `n8n-nodes-base.googleSheetsTool` — tool callable by the agent; appends one expense row.
- **Configuration (interpreted):**
  - Operation: `append`
  - Sheet: current month name in English (`MMMM yyyy`)
  - Column mapping (USER_ENTERED):
    - Date: `={{ $fromAI('Date', '', 'string') }}`
    - Category: `={{ $fromAI('Category', '', 'string') }}`
    - Description: `={{ $fromAI('Description', '', 'string') }}`
    - Amount: `={{ $fromAI('Amount', '', 'string') }}`
    - Payment_Method: `={{ $fromAI('Payment_Method', '', 'string') }}`
  - Tool description tells the agent expected parameters and allowed categories.
- **Connections:**
  - **ai_tool →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - Sheet for the month doesn’t exist → append fails (“Unable to find sheet”).
  - Column header mismatch (e.g., sheet uses `Payment Method` but node uses `Payment_Method`) → data may map incorrectly or fail depending on configuration.
  - Amount stored as string; if spreadsheet expects numeric, consider converting to number or enforcing numeric formatting.
  - OAuth2 token expired/invalid.
- **Version notes:** `typeVersion 4.7` (Google Sheets node v4+ has different field mapping behaviors than older versions).

#### Node: get_expense
- **Type / role:** `n8n-nodes-base.googleSheetsTool` — tool callable by agent; reads expense data (current month sheet).
- **Configuration (interpreted):**
  - Operation: (implicit “read/get”; parameters show sheet + document only)
  - Sheet: `={{ $now.setLocale('en').toFormat('MMMM yyyy') }}`
- **Connections:**
  - **ai_tool →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - Same sheet-name existence issue.
  - Large sheets may increase response size; agent responses could exceed Telegram limits (handled later via chunking).
- **Version notes:** `typeVersion 4.7`.

#### Node: delete_expense
- **Type / role:** `n8n-nodes-base.googleSheetsTool` — tool callable by agent; deletes last N rows.
- **Configuration (interpreted):**
  - Operation: `delete`
  - Sheet: current month
  - Number to delete: `={{ $fromAI('Number_of_Rows_to_Delete', '', 'number') }}`
- **Connections:**
  - **ai_tool →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - If AI returns non-numeric / empty → deletion fails or deletes 0 depending on node behavior.
  - Deletes only from bottom; cannot delete “middle” rows (noted in sticky info and system message).
  - Risk: user asks to delete “yesterday’s lunch” but it isn’t last row; agent must refuse/clarify.
- **Version notes:** `typeVersion 4.7`.

#### Node: add_income
- **Type / role:** `n8n-nodes-base.googleSheetsTool` — tool callable by agent; appends one income row.
- **Configuration (interpreted):**
  - Operation: `append`
  - Sheet: `Income Log`
  - Column mapping:
    - Date, Category, Description, Amount, Account all from `$fromAI(...)` as strings
- **Connections:**
  - **ai_tool →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - “Income Log” sheet must exist and headers must match node mapping.
  - Amount stored as string; numeric formatting may require sheet formatting or conversion.
- **Version notes:** `typeVersion 4.7`.

#### Node: get_income
- **Type / role:** `n8n-nodes-base.googleSheetsTool` — tool callable by agent; reads “Income Log”.
- **Configuration (interpreted):**
  - Sheet: `Income Log`
- **Connections:**
  - **ai_tool →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - Large log size could create large tool outputs; agent summarization may be needed.
- **Version notes:** `typeVersion 4.7`.

---

### Block D — Agent Utility Tools (Calculator + Think)
**Overview:** Provides the agent with calculation and structured reasoning tools to improve consistency for totals and multi-step operations (especially deletion logic).  
**Nodes involved:** `Calculator`, `Think`

#### Node: Calculator
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCalculator` — arithmetic tool for sums/averages/totals.
- **Configuration:** Default.
- **Connections:**
  - **ai_tool →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - LLM might pass malformed expressions; tool may error or return unexpected results.
- **Version notes:** `typeVersion 1`.

#### Node: Think
- **Type / role:** `@n8n/n8n-nodes-langchain.toolThink` — reasoning scratchpad tool used for multi-step logic.
- **Configuration:** Default.
- **Connections:**
  - **ai_tool →** `AI Agent - Expense Tracker`
- **Edge cases / failures:**
  - Mainly improves reasoning; doesn’t directly mutate data. Low risk.
- **Version notes:** `typeVersion 1.1`.

---

### Block E — Response Formatting, Chunking, and Telegram Send
**Overview:** Converts agent output from markdown-like formatting to Telegram HTML, splits responses to stay under Telegram limits, and sends to the user.  
**Nodes involved:** `Markdown to HTML Formatter`, `Send Telegram Message`

#### Node: Markdown to HTML Formatter
- **Type / role:** `n8n-nodes-base.code` — custom JS formatter and message chunker.
- **Configuration (interpreted):**
  - Converts:
    - Triple backticks → `<pre>...</pre>`
    - Inline backticks → `<code>...</code>`
    - Markdown links `[label](url)` → `<a href="url">label</a>`
    - `**bold**` → `<b>...</b>`
    - `*italic*` → `<i>...</i>`
  - Escapes HTML safely (prevents Telegram HTML injection).
  - Chunking:
    - Telegram hard limit: 4096 chars
    - Uses `SAFE_BUDGET = 4000` to reduce risk of overshoot
    - Splits by paragraphs, then lines, then hard chunks.
  - Input field selection: uses `j.output ?? j.message ?? j.text ?? j.content ?? ''`
  - Output per chunk: sets `json.output` and `json.message` to the chunk, adds `part_index`, `parts_total`.
- **Connections:**
  - **Input ←** `AI Agent - Expense Tracker` (success output)
  - **Output →** `Send Telegram Message`
- **Edge cases / failures:**
  - Some Markdown edge cases (nested formatting, underscores, complex links) not supported.
  - Regex limitations: italic regex may behave unexpectedly with asterisks in numbers/text.
  - If agent outputs already-HTML, it will be escaped (by design).
- **Version notes:** `typeVersion 2` (Code node v2 runtime differences vs v1).

#### Node: Send Telegram Message
- **Type / role:** `n8n-nodes-base.telegram` — sends final response to Telegram chat.
- **Configuration (interpreted):**
  - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
  - Text: `={{ $json.message }}`
  - Parse mode: `HTML`
  - Attribution disabled: `appendAttribution: false`
- **Connections:**
  - **Input ←** `Markdown to HTML Formatter`
- **Edge cases / failures:**
  - If chat ID lookup fails (e.g., node rename or missing execution context), message won’t send.
  - Telegram HTML parse errors if malformed tags slip through (formatter is designed to reduce this risk).
  - Rate limits when chunking produces many messages.
- **Version notes:** `typeVersion 1.2`.

---

### Block F — Error Handling Path
**Overview:** If the AI Agent errors (tool failures, model errors not recovered by retry), a generic error message is sent to the same Telegram chat.  
**Nodes involved:** `Send Error Message`

#### Node: Send Error Message
- **Type / role:** `n8n-nodes-base.telegram` — fallback error notification.
- **Configuration (interpreted):**
  - Text:  
    `Error occurred. Please try again!` + support hint
  - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
  - Attribution disabled.
- **Connections:**
  - **Input ←** `AI Agent - Expense Tracker` (error output)
- **Edge cases / failures:**
  - Same chat ID dependency as the normal send node.
  - If Telegram credentials fail, user receives nothing (consider adding logging).
- **Version notes:** `typeVersion 1.2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram messages | — | Typing..., AI Agent - Expense Tracker |  |
| Typing... | telegram | UX: show typing action | Telegram Trigger | — |  |
| AI Agent - Expense Tracker | @n8n/n8n-nodes-langchain.agent | Parse message, call tools, craft response | Telegram Trigger; (AI) Google Gemini Chat Model; (AI) Simple Memory; (AI tools) all tools | Markdown to HTML Formatter (success); Send Error Message (error) | 💡 HOW IT WORKS: Telegram-based AI expense tracker; dynamic monthly sheets via `$now.setLocale('en').toFormat('MMMM yyyy')`; commands; typing indicator; auto-splitting; delete limitation (only last N rows). |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM provider for agent | — | AI Agent - Expense Tracker (ai_languageModel) | ⚙️ CONFIGURATION: create Gemini API key at https://ai.google.dev and assign to this node. |
| Simple Memory | memoryBufferWindow | Per-chat memory keyed by Telegram chat id | — | AI Agent - Expense Tracker (ai_memory) |  |
| add_expense | googleSheetsTool | Tool: append one expense row to monthly sheet | — (tool invoked by agent) | AI Agent - Expense Tracker (ai_tool) | ⚙️ CONFIGURATION: set `YOUR_GOOGLE_SHEET_ID_HERE` in all Google Sheets nodes; assign Google Sheets OAuth2 credential. |
| get_expense | googleSheetsTool | Tool: read expenses from monthly sheet | — (tool invoked by agent) | AI Agent - Expense Tracker (ai_tool) | ⚙️ CONFIGURATION: set `YOUR_GOOGLE_SHEET_ID_HERE` in all Google Sheets nodes; assign Google Sheets OAuth2 credential. |
| delete_expense | googleSheetsTool | Tool: delete last N rows from monthly sheet | — (tool invoked by agent) | AI Agent - Expense Tracker (ai_tool) | 💡 HOW IT WORKS: delete limitation—can only delete last N rows. |
| add_income | googleSheetsTool | Tool: append one income row to “Income Log” | — (tool invoked by agent) | AI Agent - Expense Tracker (ai_tool) | ⚙️ CONFIGURATION: set `YOUR_GOOGLE_SHEET_ID_HERE` in all Google Sheets nodes; ensure “Income Log” sheet exists with correct headers. |
| get_income | googleSheetsTool | Tool: read “Income Log” | — (tool invoked by agent) | AI Agent - Expense Tracker (ai_tool) | ⚙️ CONFIGURATION: set `YOUR_GOOGLE_SHEET_ID_HERE` in all Google Sheets nodes. |
| Calculator | toolCalculator | Tool: arithmetic for totals/summaries | — (tool invoked by agent) | AI Agent - Expense Tracker (ai_tool) |  |
| Think | toolThink | Tool: multi-step reasoning (esp. deletes) | — (tool invoked by agent) | AI Agent - Expense Tracker (ai_tool) |  |
| Markdown to HTML Formatter | code | Convert markdown-like output to Telegram HTML + chunking | AI Agent - Expense Tracker (success) | Send Telegram Message | 💡 HOW IT WORKS: auto-splits long responses; uses Telegram HTML parse mode. |
| Send Telegram Message | telegram | Deliver formatted reply to user | Markdown to HTML Formatter | — |  |
| Send Error Message | telegram | Deliver generic error on agent failure | AI Agent - Expense Tracker (error) | — |  |
| 📋 Prerequisites | stickyNote | Documentation note | — | — | 📋 PREREQUISITES: Google account + Sheets OAuth2; Telegram bot via @BotFather; Gemini API key via https://ai.google.dev; sheet structure (monthly “MMMM yyyy”, and “Income Log”); categories/payment methods; Creator: Andi Sakti; LinkedIn: https://www.linkedin.com/in/andi-sakti-cpos-7922b4184/ |
| ⚙️ Configuration | stickyNote | Documentation note | — | — | ⚙️ CONFIGURATION: create credentials; set Document ID (docs.google.com/spreadsheets/d/THIS_PART/edit); assign credentials to nodes; test and activate; ALL 5 Sheets nodes need Document ID. |
| 💡 Important Info | stickyNote | Documentation note | — | — | 💡 HOW IT WORKS: commands, features, dynamic sheet naming expression, delete limitation. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Telegram Trigger**
   - Node: *Telegram Trigger*
   - Updates: **message**
   - Credentials: Telegram Bot Token (create via **@BotFather**)
3. **Add “Typing…” node**
   - Node: *Telegram* (not trigger)
   - Operation: **Send Chat Action**
   - Chat ID: `{{$json.message.chat.id}}`
   - Connect: `Telegram Trigger` → `Typing...`
4. **Add Google Gemini Chat Model**
   - Node: *Google Gemini Chat Model*
   - Temperature: **0.6**
   - Credentials: Google Gemini API key from **https://ai.google.dev**
5. **Add Simple Memory**
   - Node: *Simple Memory* (Buffer Window)
   - Session ID type: **Custom Key**
   - Session key expression: `{{$json.message.chat.id}}`
6. **Add Google Sheets Tool nodes (5 nodes total)** and set credentials
   - Create Google Sheets OAuth2 credential (Google Cloud Console OAuth client) and attach to each Sheets node.
   - Replace `YOUR_GOOGLE_SHEET_ID_HERE` with your spreadsheet ID in *all 5 nodes*.
   - Ensure your spreadsheet has:
     - Monthly sheets named exactly like **“February 2026”** (English month) with headers: `Date | Category | Description | Amount | Payment_Method`
     - A sheet named **“Income Log”** with headers: `Date | Category | Description | Amount | Account`
   1) **add_expense**
      - Operation: **Append**
      - Sheet name: `{{$now.setLocale('en').toFormat('MMMM yyyy')}}`
      - Set columns (USER_ENTERED recommended):
        - Date: `{{$fromAI('Date','','string')}}`
        - Category: `{{$fromAI('Category','','string')}}`
        - Description: `{{$fromAI('Description','','string')}}`
        - Amount: `{{$fromAI('Amount','','string')}}`
        - Payment_Method: `{{$fromAI('Payment_Method','','string')}}`
      - Add a clear tool description indicating required parameters.
   2) **get_expense**
      - Operation: read/get rows (configure “Get Many”/equivalent in your n8n version)
      - Sheet name: `{{$now.setLocale('en').toFormat('MMMM yyyy')}}`
   3) **delete_expense**
      - Operation: **Delete**
      - Sheet name: `{{$now.setLocale('en').toFormat('MMMM yyyy')}}`
      - Number to delete: `{{$fromAI('Number_of_Rows_to_Delete','','number')}}`
   4) **add_income**
      - Operation: **Append**
      - Sheet name: `Income Log`
      - Columns:
        - Date, Category, Description, Amount, Account using `$fromAI(...)` (strings)
   5) **get_income**
      - Operation: read/get rows
      - Sheet name: `Income Log`
7. **Add utility tools**
   - Node: *Calculator* tool
   - Node: *Think* tool
8. **Add AI Agent**
   - Node: *AI Agent* (LangChain Agent)
   - Text input: `{{$json.message.text}}`
   - Prompt type: **Define**
   - System message: include
     - Tool list and rules (“call tool first”, multiple items → multiple tool calls)
     - Category/payment method mapping
     - Amount parsing rules (k/rb/jt)
     - Workflows for record/view/delete
     - Response templates and commands
     - Current time injection (optional): `{{$now.format('cccc DD HH:mm')}}`
   - Error handling: set to **Continue with error output** (or equivalent)
   - Retry on fail: enabled; wait 5000ms
9. **Wire the AI Agent’s AI ports**
   - Connect *Google Gemini Chat Model* → AI Agent (**ai_languageModel**)
   - Connect *Simple Memory* → AI Agent (**ai_memory**)
   - Connect each tool node (**add_expense, get_expense, delete_expense, add_income, get_income, Calculator, Think**) → AI Agent (**ai_tool**)
10. **Wire the main execution path**
   - Connect `Telegram Trigger` → `AI Agent`
   - (Keep `Telegram Trigger` → `Typing...` in parallel)
11. **Add “Markdown to HTML Formatter” (Code node)**
   - Node: *Code*
   - Paste JS logic that:
     - Escapes HTML
     - Converts basic markdown to Telegram HTML (`<b>`, `<i>`, `<code>`, `<pre>`, `<a href=...>`)
     - Chunks output to ~4000 chars
     - Outputs one item per chunk with `json.message` set
   - Connect: `AI Agent (success)` → `Markdown to HTML Formatter`
12. **Add Send Telegram Message**
   - Node: *Telegram*
   - Operation: **Send Message**
   - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`
   - Text: `{{$json.message}}`
   - Parse mode: **HTML**
   - Disable attribution if desired
   - Connect: `Markdown to HTML Formatter` → `Send Telegram Message`
13. **Add Send Error Message**
   - Node: *Telegram*
   - Operation: **Send Message**
   - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`
   - Text: a generic failure message
   - Connect: `AI Agent (error output)` → `Send Error Message`
14. **Test**
   - Use “Test workflow”, send a Telegram message like:
     - “Lunch 25k cash”
     - “Salary 5jt transfer”
     - “/today”
   - Verify sheet entries and bot replies.
15. **Activate workflow** once stable.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Gemini API key source | https://ai.google.dev |
| Spreadsheet Document ID location | `docs.google.com/spreadsheets/d/THIS_PART/edit` |
| Creator credit | Andi Sakti |
| LinkedIn | https://www.linkedin.com/in/andi-sakti-cpos-7922b4184/ |
| Dynamic monthly sheet naming expression | `$now.setLocale('en').toFormat('MMMM yyyy')` |
| Delete limitation | Only the **last N rows** can be deleted automatically; middle rows require manual deletion in Sheets. |