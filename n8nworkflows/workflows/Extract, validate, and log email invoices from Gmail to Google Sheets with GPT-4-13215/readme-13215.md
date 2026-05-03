Extract, validate, and log email invoices from Gmail to Google Sheets with GPT-4

https://n8nworkflows.xyz/workflows/extract--validate--and-log-email-invoices-from-gmail-to-google-sheets-with-gpt-4-13215


# Extract, validate, and log email invoices from Gmail to Google Sheets with GPT-4

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Inbox2Ledger  
**Title (given):** Extract, validate, and log email invoices from Gmail to Google Sheets with GPT-4  
**Purpose:** Collect emails from Gmail within a date window, filter down to finance-relevant messages using an AI guardrail plus subject keyword checks, extract invoice/receipt fields using an LLM “OCR-style” agent that outputs normalized JSON, validate the extraction, enrich it with finance categorization rules and unique case IDs, then append the structured record into a Google Sheets ledger.

### 1.1 Input Reception (date window + email fetch)
- Entry is a **Form Trigger** where the user provides a “Date till which…” value.
- Gmail retrieves all inbox emails **received after** (date - 1 day at 00:00:00).

### 1.2 Finance relevance filtering (AI guardrail + keyword filter)
- An AI **Topical Alignment guardrail** checks whether the email is about an invoice/receipt/transaction.
- If guardrail passes, a **keyword filter** on subject further narrows to invoice/receipt/bill/payment confirmations.

### 1.3 AI extraction (structured JSON)
- An LLM agent extracts invoice fields from email body text, returning **JSON only**, with normalized dates and numeric amounts.

### 1.4 Validation and error gating
- Code node parses/validates JSON and required fields, emitting either an error payload or validated data.
- IF node allows only non-error items to proceed.

### 1.5 Finance enrichment + ledger logging
- Code node assigns an **expense category** via rule map + fallbacks and generates `case_id` and timestamps.
- Google Sheets appends a row to the “Invoices” sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception: date form → Gmail fetch
**Overview:** Collects a user-selected date and fetches Gmail messages from INBOX received after (date - 1 day). This creates the batch of emails to be filtered and processed.  
**Nodes involved:**  
- Enter Date till which you want email to be fetched  
- Get Email Content  

#### Node: **Enter Date till which you want email to be fetched**
- **Type / role:** `n8n-nodes-base.formTrigger` — manual entry point via hosted n8n form.
- **Configuration (interpreted):**
  - Form title: “AI Financial Mail Detector and Summarizer”
  - One required field: date labeled **“Date till which you want your mails to be summarized”**
- **Key variables / outputs:**
  - Produces `$json["Date till which you want your mails to be summarized"]` as a date value.
- **Connections:**
  - Output → **Get Email Content**
- **Potential failures / edge cases:**
  - User submits invalid/empty date (mitigated by `requiredField: true` but still depends on form UI).
  - Date timezone ambiguity (date-only inputs can behave differently depending on instance locale).

#### Node: **Get Email Content**
- **Type / role:** `n8n-nodes-base.gmail` — fetches emails from Gmail.
- **Operation:** `getAll` with `returnAll: true` (fetch all matching messages).
- **Configuration (interpreted):**
  - Label filter: `INBOX`
  - Read status: `both` (read + unread)
  - Received after is computed by expression:
    - Takes the submitted date, subtracts **1 day**, then sets time to **T00:00:00** and uses ISO date prefix.
- **Key expression:**
  - `receivedAfter`:
    - Uses `new Date($json['Date till which you want your mails to be summarized'])`
    - `d.setDate(d.getDate() - 1)`
    - Returns `YYYY-MM-DDT00:00:00`
- **Credentials:** Gmail OAuth2 (`Gmail account 2`)
- **Connections:**
  - Output → **Guardrail: Is Finance?**
- **Potential failures / edge cases:**
  - Gmail OAuth expired/invalid scopes.
  - Large inbox volumes can cause long executions or rate limits (especially with `returnAll: true`).
  - The node is named “Get Email Content” but the guardrail later references `item.json.text`; if Gmail returns `textHtml`/`textPlain` depending on configuration, field availability must be confirmed.
  - The expression uses a date from the form; if the form returns a string in unexpected format, `new Date()` parsing may yield `Invalid Date`.

---

### Block 2 — Finance relevance filtering: AI guardrail → IF pass → subject keyword filter
**Overview:** Filters out non-finance emails using an AI topical alignment check, then applies a deterministic subject keyword filter to further reduce false positives.  
**Nodes involved:**  
- OpenAI Chat Model  
- Guardrail: Is Finance?  
- IF (Guardrail Passed)  
- Filter Finance Keywords  

#### Node: **OpenAI Chat Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides a chat LLM for LangChain-based nodes.
- **Configuration (interpreted):**
  - Model: `gpt-4.1`
  - Responses API disabled
- **Credentials:** OpenAI (`OpenAi account`)
- **Connections:**
  - Used as an **AI language model input** by **Guardrail: Is Finance?**
- **Potential failures / edge cases:**
  - OpenAI credential issues, quota, model availability.
  - Higher latency/timeouts if processing many emails.

#### Node: **Guardrail: Is Finance?**
- **Type / role:** `@n8n/n8n-nodes-langchain.guardrails` — applies guardrails (here: topical alignment) to decide if content is in scope.
- **Configuration (interpreted):**
  - Input text combines email body and subject:
    - `{{ $('Get Email Content').item.json.text }} and {{ $json.headers.subject }}`
  - Guardrail: **Topical Alignment**
    - Prompt defines on-topic vs off-topic finance criteria.
    - Threshold: `0.5` (moderate strictness)
- **Key variables / outputs:**
  - Outputs a `checks` object; later logic uses `checks.triggered` (boolean).
- **Connections:**
  - Main output → **IF (Guardrail Passed)**
  - AI model input from **OpenAI Chat Model**
- **Potential failures / edge cases:**
  - Missing fields: if Gmail output does not include `text` or `headers.subject`, the prompt may be incomplete and misclassify.
  - Threshold too low/high: too low allows newsletters; too high drops legitimate invoices with minimal text.
  - LLM refusal or transient API errors cause guardrail node to fail execution.

#### Node: **IF (Guardrail Passed)**
- **Type / role:** `n8n-nodes-base.if` — routes only items that passed the guardrail.
- **Configuration (interpreted):**
  - Condition checks boolean operation “false” on:
    - `{{ $json.checks.triggered }}`
  - Meaning: proceed when **checks.triggered is false** (i.e., guardrail did not trigger a violation/off-topic condition).
- **Connections:**
  - True branch → **Filter Finance Keywords**
  - (False branch is unused; non-finance emails are dropped)
- **Potential failures / edge cases:**
  - If `checks.triggered` is undefined, loose validation may treat it unexpectedly; confirm guardrail output structure in your n8n version.

#### Node: **Filter Finance Keywords**
- **Type / role:** `n8n-nodes-base.filter` — deterministic subject-based filter for finance terms.
- **Configuration (interpreted):**
  - OR conditions: subject contains any of:
    - “Invoice”
    - “Receipt”
    - “Bill”
    - “Payment Confirmation”
    - “Payment”
  - Subject referenced as:
    - `{{ $('Get Email Content').item.json.subject }}`
- **Connections:**
  - Output → **AI Agent (Email OCR)**
- **Potential failures / edge cases:**
  - Case sensitivity: “invoice” lowercase won’t match unless Gmail subject casing aligns (filter uses `contains` with default behavior; in n8n it is case-sensitive unless configured otherwise).
  - Non-English subjects or vendor-specific terms (e.g., “Tax Invoice”, “Order confirmation”) may be missed.
  - References `Get Email Content...subject`; if Gmail node provides `headers.subject` instead, this filter may not match anything until adjusted.

---

### Block 3 — AI Extraction: OCR-style JSON extraction from email body
**Overview:** Uses an LLM agent to extract invoice/receipt fields from the email body and return a strict JSON object.  
**Nodes involved:**  
- gpt 4o mini  
- AI Agent (Email OCR)  

#### Node: **gpt 4o mini**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider for the extraction agent.
- **Configuration (interpreted):**
  - Model: `gpt-4o-mini`
  - Temperature: `0.2` (low variance)
  - Response format: `json_object` (strongly nudges valid JSON)
- **Credentials:** OpenAI (`OpenAi account`)
- **Connections:**
  - Used as **AI language model input** by **AI Agent (Email OCR)**
- **Potential failures / edge cases:**
  - Response format compliance is not guaranteed if prompt is long/complex or email text is malformed.
  - Model availability or account rate limits.

#### Node: **AI Agent (Email OCR)**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that prompts the model to extract structured data.
- **Configuration (interpreted):**
  - Input text: `{{ $('Get Email Content').item.json.text }}`
  - System message instructs:
    - Return **ONLY** a JSON object
    - Date normalized to `YYYY-MM-DD`
    - Amounts numeric (currency symbols removed)
  - Required JSON keys:
    - `vendor_name`, `invoice_date`, `invoice_id`, `total_amount`, `tax_amount`, `currency`, `items_summary`, `vendor_tax_id`
  - `hasOutputParser: true` (expects structured output handling)
  - **continueOnFail: true** (workflow continues even if this node errors; downstream validation handles it)
- **Connections:**
  - Main output → **Validate Extraction**
  - AI model input from **gpt 4o mini**
- **Potential failures / edge cases:**
  - Email bodies that are HTML-only or missing `text` field reduce extraction quality.
  - Multi-currency, partial payments, or statements may not map cleanly to one `total_amount`.
  - If the agent outputs JSON but with trailing text, downstream JSON parsing can fail.
  - If extraction fails, node may output an `error` object which is explicitly handled later.

---

### Block 4 — Validation and gating: parse → validate → proceed only if clean
**Overview:** Ensures the AI output is valid JSON and contains required fields; blocks invalid items from being logged as invoices.  
**Nodes involved:**  
- Validate Extraction  
- Check for Errors  

#### Node: **Validate Extraction**
- **Type / role:** `n8n-nodes-base.code` — parses agent output, validates required fields and amount type, emits standardized error payloads.
- **Configuration (interpreted):**
  - Runs once per item (`runOnceForEachItem`)
  - Logic:
    1. If `$json.error` exists → emit `error_occurred: true`, `AI_EXTRACTION_FAILED`
    2. Parse `$json.output` as JSON if string, else use as object
       - If parse fails → `JSON_PARSE_ERROR`
    3. Require fields: `vendor_name`, `total_amount`, `invoice_date`
       - Missing → `MISSING_FIELDS` + include `extracted_data`
    4. Validate `total_amount` is numeric → else `INVALID_AMOUNT`
    5. If all good → `error_occurred: false`, `validated_data: data`
- **Connections:**
  - Output → **Check for Errors**
- **Potential failures / edge cases:**
  - If the agent returns structured data in another property than `output`, parsing will fail.
  - Date normalization is not validated beyond presence; invalid dates may still pass.
  - Numeric validation uses `parseFloat`; strings like `"12,34"` (comma decimal) may parse incorrectly.

#### Node: **Check for Errors**
- **Type / role:** `n8n-nodes-base.if` — allows only validated items through.
- **Configuration (interpreted):**
  - Condition uses boolean operation “false” on:
    - `{{ $json.error_occurred }}`
  - Meaning: proceed only when `error_occurred` is **false**.
- **Connections:**
  - True branch → **Apply Finance Rules**
  - (False branch unused; errors are silently dropped unless you add logging/alerts)
- **Potential failures / edge cases:**
  - If `error_occurred` is missing, loose type validation could behave unexpectedly.

---

### Block 5 — Enrichment and logging: categorize → append to Google Sheets
**Overview:** Categorizes expenses using keyword rules and fallbacks, generates case IDs, adds timestamps, then appends a row to Google Sheets.  
**Nodes involved:**  
- Apply Finance Rules  
- Log to Invoices Sheet  

#### Node: **Apply Finance Rules**
- **Type / role:** `n8n-nodes-base.code` — enriches validated invoice data with expense categorization and identifiers.
- **Configuration (interpreted):**
  - Builds `combinedText` from normalized fields (lowercased):  
    `vendor_name, merchant_name, description, notes, expense_type`
  - Applies `CATEGORY_MAP` keyword lists → sets `expense_category`
  - Fallbacks:
    - MCC-based fallback if `data.mcc` exists:
      - startsWith “5” → Travel and meals
      - startsWith “7” → Professional services
    - Vendor fallback (e.g., “airbnb”, “amazon”, “flipkart”)
    - Final default: “Other expenses”
  - Adds:
    - `case_id`: `INV-{timestamp}-{random6}`
    - `processed_at`: ISO timestamp
- **Connections:**
  - Output → **Log to Invoices Sheet**
- **Potential failures / edge cases:**
  - Output schema mismatch: Google Sheets node expects a `gl_category` column, but this code produces `expense_category` (unless the sheet mapping is tolerant or you adjust mapping/column name).
  - Keyword map is English-centric; non-English vendors/descriptions may default to “Other expenses”.
  - `Date.now()` uniqueness is good but not collision-proof at very high throughput (rare).

#### Node: **Log to Invoices Sheet**
- **Type / role:** `n8n-nodes-base.googleSheets` — appends processed invoice row to a ledger sheet.
- **Operation:** `append`
- **Configuration (interpreted):**
  - Document: **Finance OCR** (Spreadsheet ID: `1pHyqJsfwCIL-XrsWnE0WZsJPxtrd7DeXEGbrKMU_XJk`)
  - Sheet: **Invoices** (gid `1260157166`)
  - Mapping mode: Auto-map input data to sheet columns (schema explicitly listed)
  - Retry on fail: enabled; wait between tries: 2000 ms
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account 3`)
- **Connections:**
  - Terminal node (no further outputs)
- **Potential failures / edge cases:**
  - OAuth scope/permission errors (no edit access to sheet).
  - Column mismatch:
    - Sheet schema includes `gl_category` and `approval_status`, `timestamp`; upstream data does not explicitly set these.
    - Upstream provides `processed_at` and `case_id`, and invoice fields; you may need a Set/Code node to map `expense_category → gl_category`, and add `timestamp`.
  - Google Sheets rate limits on high volume runs.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Enter Date till which you want email to be fetched | n8n-nodes-base.formTrigger | Manual entry point: collect cutoff date | — | Get Email Content | Inbox2Ledger is an end-to-end **n8n template** that turns a noisy finance inbox into a clean, structured ledger. |
| Get Email Content | n8n-nodes-base.gmail | Fetch Gmail messages from INBOX after computed date | Enter Date till which you want email to be fetched | Guardrail: Is Finance? | # Get all emails; Connect Gmail and fetch incoming messages or selected folders; batch or trigger-based retrieval ensures full inbox coverage for automatic processing. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for guardrails | — | Guardrail: Is Finance? (as AI model) | # Filter finance emails; Apply AI guardrails and keyword filters to isolate invoices, receipts, and payments; reduce false positives and route only finance messages. |
| Guardrail: Is Finance? | @n8n/n8n-nodes-langchain.guardrails | AI topical alignment filter for finance relevance | Get Email Content + OpenAI Chat Model (AI model) | IF (Guardrail Passed) | # Filter finance emails; Apply AI guardrails and keyword filters to isolate invoices, receipts, and payments; reduce false positives and route only finance messages. |
| IF (Guardrail Passed) | n8n-nodes-base.if | Route only emails that pass guardrail | Guardrail: Is Finance? | Filter Finance Keywords | # Filter finance emails; Apply AI guardrails and keyword filters to isolate invoices, receipts, and payments; reduce false positives and route only finance messages. |
| Filter Finance Keywords | n8n-nodes-base.filter | Subject keyword filter for invoice/receipt/payment | IF (Guardrail Passed) | AI Agent (Email OCR) | # Filter finance emails; Apply AI guardrails and keyword filters to isolate invoices, receipts, and payments; reduce false positives and route only finance messages. |
| gpt 4o mini | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for extraction agent | — | AI Agent (Email OCR) (as AI model) | # Extract details and log; Use LLM/OCR agent to extract vendor, date, invoice_id, amounts; validate entries, categorize expenses, and append structured rows to Google Sheets. |
| AI Agent (Email OCR) | @n8n/n8n-nodes-langchain.agent | Extract invoice fields as JSON from email text | Filter Finance Keywords + gpt 4o mini (AI model) | Validate Extraction | # Extract details and log; Use LLM/OCR agent to extract vendor, date, invoice_id, amounts; validate entries, categorize expenses, and append structured rows to Google Sheets. |
| Validate Extraction | n8n-nodes-base.code | Parse/validate JSON, required fields, amount type | AI Agent (Email OCR) | Check for Errors | # Extract details and log; Use LLM/OCR agent to extract vendor, date, invoice_id, amounts; validate entries, categorize expenses, and append structured rows to Google Sheets. |
| Check for Errors | n8n-nodes-base.if | Allow only validated items through | Validate Extraction | Apply Finance Rules | # Extract details and log; Use LLM/OCR agent to extract vendor, date, invoice_id, amounts; validate entries, categorize expenses, and append structured rows to Google Sheets. |
| Apply Finance Rules | n8n-nodes-base.code | Categorize expense, generate case_id, timestamps | Check for Errors | Log to Invoices Sheet | # Extract details and log; Use LLM/OCR agent to extract vendor, date, invoice_id, amounts; validate entries, categorize expenses, and append structured rows to Google Sheets. |
| Log to Invoices Sheet | n8n-nodes-base.googleSheets | Append invoice row to Google Sheets ledger | Apply Finance Rules | — | # Extract details and log; Use LLM/OCR agent to extract vendor, date, invoice_id, amounts; validate entries, categorize expenses, and append structured rows to Google Sheets. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block | — | — | # Get all emails; Connect Gmail and fetch incoming messages or selected folders; batch or trigger-based retrieval ensures full inbox coverage for automatic processing. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment block | — | — | 👉 [Demo & Setup Video](https://drive.google.com/file/d/1OeDKjgO0a9yXMMShGTO08JGJDzvc9wl8/view?usp=sharing) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment block | — | — | # Filter finance emails; Apply AI guardrails and keyword filters to isolate invoices, receipts, and payments; reduce false positives and route only finance messages. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment block | — | — | # Extract details and log; Use LLM/OCR agent to extract vendor, date, invoice_id, amounts; validate entries, categorize expenses, and append structured rows to Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it **Inbox2Ledger** (or your preferred name).
- Set execution order to **v1** (Workflow Settings → Execution).

2) **Add the form entry node**
- Add node: **Form Trigger**  
  - Name: “Enter Date till which you want email to be fetched”
  - Form title: “AI Financial Mail Detector and Summarizer”
  - Add one field:
    - Type: **Date**
    - Label: “Date till which you want your mails to be summarized”
    - Required: **true**

3) **Add Gmail email fetch**
- Add node: **Gmail**
  - Name: “Get Email Content”
  - Operation: **Get Many / Get All** (GetAll)
  - Return all: **true**
  - Filters:
    - Label IDs: `INBOX`
    - Read status: `both`
    - Received after: set an **expression** that subtracts 1 day from the submitted date and returns `YYYY-MM-DDT00:00:00` (same logic as described in section 2).
- **Credentials:** Create/attach **Gmail OAuth2** credentials with permission to read mail.
- Connect: **Form Trigger → Gmail**

4) **Add OpenAI model for guardrail**
- Add node: **OpenAI Chat Model** (LangChain)
  - Name: “OpenAI Chat Model”
  - Model: `gpt-4.1`
- **Credentials:** Create/attach **OpenAI API** credentials.

5) **Add guardrail node**
- Add node: **Guardrails** (LangChain Guardrails)
  - Name: “Guardrail: Is Finance?”
  - Text expression: combine email body + subject (ensure you reference the correct Gmail fields in your environment).
  - Enable **Topical Alignment** with:
    - Prompt describing finance on-topic/off-topic scope
    - Threshold: `0.5`
- Connect:
  - Main: **Gmail → Guardrail**
  - AI Language Model input: **OpenAI Chat Model → Guardrail** (AI connection)

6) **Add IF gate for guardrail pass**
- Add node: **IF**
  - Name: “IF (Guardrail Passed)”
  - Condition: proceed when guardrail did **not** trigger
    - Boolean check on `checks.triggered` equals **false** (configure using the boolean operators).
- Connect: **Guardrail → IF**

7) **Add subject keyword filter**
- Add node: **Filter**
  - Name: “Filter Finance Keywords”
  - Conditions: OR
    - Subject contains “Invoice”
    - Subject contains “Receipt”
    - Subject contains “Bill”
    - Subject contains “Payment Confirmation”
    - Subject contains “Payment”
  - Use an expression pointing to the subject field produced by Gmail.
- Connect: **IF (true) → Filter**

8) **Add OpenAI model for extraction**
- Add node: **OpenAI Chat Model** (LangChain)
  - Name: “gpt 4o mini”
  - Model: `gpt-4o-mini`
  - Temperature: `0.2`
  - Response format: `json_object`
- Reuse the same OpenAI credentials (or another, if desired).

9) **Add the AI Agent extraction node**
- Add node: **AI Agent** (LangChain Agent)
  - Name: “AI Agent (Email OCR)”
  - Text input: email body (expression from Gmail node)
  - System message: instruct strict JSON-only output and required schema (vendor, date, invoice id, amounts, currency, etc.)
  - Turn on output parsing if available (**hasOutputParser** behavior)
  - Enable **Continue On Fail** so downstream logic can handle failures.
- Connect:
  - Main: **Filter → AI Agent**
  - AI Language Model input: **gpt 4o mini → AI Agent** (AI connection)

10) **Add validation Code node**
- Add node: **Code**
  - Name: “Validate Extraction”
  - Mode: **Run Once for Each Item**
  - Implement logic:
    - If AI node errored → emit standardized error object
    - Parse JSON from AI output
    - Validate required fields and numeric amount
    - Emit `error_occurred` boolean + `validated_data` when OK
- Connect: **AI Agent → Validate Extraction**

11) **Add error gate**
- Add node: **IF**
  - Name: “Check for Errors”
  - Condition: `error_occurred` equals **false**
- Connect: **Validate Extraction → Check for Errors**

12) **Add finance categorization Code node**
- Add node: **Code**
  - Name: “Apply Finance Rules”
  - Build a keyword-based category map and fallbacks.
  - Output enriched record including:
    - expense category field (consider naming it `gl_category` to match your sheet)
    - `case_id`
    - `processed_at`
- Connect: **Check for Errors (true) → Apply Finance Rules**

13) **Add Google Sheets append**
- Add node: **Google Sheets**
  - Name: “Log to Invoices Sheet”
  - Operation: **Append**
  - Select your spreadsheet (or use:
    - Spreadsheet ID: `1pHyqJsfwCIL-XrsWnE0WZsJPxtrd7DeXEGbrKMU_XJk`
    - Sheet: “Invoices”)
  - Mapping mode: **Auto-map input data**
  - Enable retry on fail; wait 2000 ms between tries (optional but recommended)
- **Credentials:** Create/attach **Google Sheets OAuth2** credentials with edit access.
- Connect: **Apply Finance Rules → Google Sheets**

14) **(Recommended) Fix schema alignment**
- If your sheet columns include `gl_category`, add a small mapping step (Set/Code) so:
  - `gl_category = expense_category`
  - Add `timestamp = processed_at` (or current time)
  - Set `approval_status` default (e.g., “pending”) if required by your ledger process.

15) **Test**
- Run the form, pick a date, confirm:
  - Emails are fetched
  - Only finance emails pass filters
  - The AI output parses as JSON
  - A row is appended to Google Sheets

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | https://drive.google.com/file/d/1OeDKjgO0a9yXMMShGTO08JGJDzvc9wl8/view?usp=sharing |
| Template description (Inbox2Ledger end-to-end flow: fetch → guardrail → extract → validate → categorize → log) | From the workflow’s main sticky note text |
| Google Sheet used by the workflow (“Finance OCR” spreadsheet, “Invoices” sheet) | Spreadsheet: https://docs.google.com/spreadsheets/d/1pHyqJsfwCIL-XrsWnE0WZsJPxtrd7DeXEGbrKMU_XJk/edit?usp=drivesdk |

