Analyze bank statements and categorize spending with monthly Slack reports

https://n8nworkflows.xyz/workflows/analyze-bank-statements-and-categorize-spending-with-monthly-slack-reports-13087


# Analyze bank statements and categorize spending with monthly Slack reports

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow monitors Gmail for incoming bank statement emails, downloads the attached PDF statement, extracts structured transaction data using PDF Vector, computes spending and savings metrics, logs a monthly summary to Google Sheets, and posts a formatted monthly report to Slack.

**Target use cases:**
- Personal finance tracking (monthly review of spending, savings rate, anomalies)
- Small business bookkeeping (categorization + summary logging)
- Budget analysis (category totals, large transaction detection)

### 1.1 Email Monitoring & Retrieval
Detects new emails (assumed to be bank statements) and fetches the full message including attachments.

### 1.2 Statement Extraction (PDF → Structured Data)
Sends the PDF attachment to PDF Vector and requests a schema-constrained JSON extraction including categories.

### 1.3 Spending Analytics & Enrichment
Computes totals (income/spending), category breakdown, large transactions, and adds metadata (processed time).

### 1.4 Persistence (Google Sheets)
Appends a row to a Google Sheet containing period and summary metrics.

### 1.5 Reporting (Slack)
Posts a formatted summary to a specific Slack channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Email Monitoring & Retrieval

**Overview:** Watches Gmail for new messages and then retrieves the full email content with attachments so downstream nodes can process the PDF statement.

**Nodes Involved:**
- Gmail Trigger
- Get a message

#### Node: Gmail Trigger
- **Type / Role:** `n8n-nodes-base.gmailTrigger` — polling trigger that emits Gmail message metadata when new mail arrives.
- **Configuration (interpreted):**
  - Polling frequency: **every minute**
  - Includes spam/trash: **disabled**
  - No explicit query/label filter is set in the provided JSON, so it may trigger for many messages unless the credential/account has other constraints.
- **Key variables/expressions:** None (emits message objects; downstream uses `{{$json.id}}`).
- **Input / Output:**
  - **Input:** None (trigger)
  - **Output:** To **Get a message**
- **Version-specific notes:** TypeVersion **1**.
- **Potential failures / edge cases:**
  - Over-triggering if mailbox is busy (no sender/subject/label filtering).
  - Gmail API rate limits due to frequent polling.
  - OAuth token expiry/refresh issues.
  - Duplicate processing if the trigger re-sees messages (depends on Gmail Trigger state handling).

#### Node: Get a message
- **Type / Role:** `n8n-nodes-base.gmail` — retrieves the full message and downloads attachments.
- **Configuration (interpreted):**
  - Operation: **get message**
  - Message ID: `={{ $json.id }}` (from Gmail Trigger output)
  - Attachments: **downloadAttachments = true**
  - “Simple” mode: **false** (returns richer/structured Gmail data)
- **Key variables/expressions:**
  - `={{ $json.id }}`
- **Input / Output:**
  - **Input:** Gmail Trigger item
  - **Output:** To **PDF Vector - Extract Statement** (must include binary attachment fields)
- **Version-specific notes:** TypeVersion **2.2**.
- **Potential failures / edge cases:**
  - Email has **no attachments** or attachment is not a PDF (downstream extraction will fail).
  - Gmail node downloads attachments into binary properties named like `attachment_0`, `attachment_1`, etc.; workflow assumes **`attachment_0`** exists and is the statement.
  - Large attachments may hit size/time limits.

---

### Block 2 — Statement Extraction (PDF → Structured Data)

**Overview:** Sends the first downloaded attachment to PDF Vector for schema-constrained extraction of statement metadata and all transactions, including classification into predefined categories.

**Nodes Involved:**
- PDF Vector - Extract Statement
- Sticky Note1 (documentation note on categories)

#### Node: PDF Vector - Extract Statement
- **Type / Role:** `n8n-nodes-pdfvector.pdfVector` — document extraction using PDF Vector with a custom prompt and output schema.
- **Configuration (interpreted):**
  - Resource: **document**
  - Operation: **extract**
  - Input type: **file**
  - Binary property name: **`attachment_0`** (from Gmail “Get a message”)
  - Prompt requests:
    - Extract transactions with: date, description, signed amount, running balance
    - Categorize each transaction using specified rules (merchant keyword heuristics)
  - Schema (JSON Schema-like) enforces:
    - `accountNumber` (string)
    - `statementPeriod.startDate/endDate` (strings)
    - balances and totals (numbers)
    - `transactions[]` with `date`, `description`, `amount`, `category` (enum), `balance`
    - **Required:** `closingBalance`, `transactions`
    - `additionalProperties: false` (strict)
- **Key variables/expressions:** None.
- **Input / Output:**
  - **Input:** Binary PDF in `attachment_0`
  - **Output:** JSON under `$json.data` (as referenced by the Code node)
  - **Output connection:** To **Analyze Spending**
- **Version-specific notes:** TypeVersion **1**; node requires PDF Vector credentials configured.
- **Potential failures / edge cases:**
  - If the attachment property name differs (e.g., not `attachment_0`), extraction fails.
  - Statement formats vary; schema strictness (`additionalProperties: false`) can cause extraction rejection if the model tries to output extra fields.
  - Date formats may be inconsistent (string parsing not enforced later).
  - Categorization errors if merchant strings don’t match provided rules.
  - API timeouts for long statements or multi-page PDFs.

#### Sticky Note1 (Spending Categories)
- **Type / Role:** `n8n-nodes-base.stickyNote` — documentation only.
- **Content (preserved):**
  - “📊 Spending Categories: Income, Utilities, Groceries, Dining, Shopping, Transport, Subscriptions, Healthcare, Entertainment, Transfers”
- **Note:** The actual PDF Vector schema includes **Transfer** (singular) and also includes **Travel** and **Other**; sticky note list is not fully aligned with the schema enum.

---

### Block 3 — Spending Analytics & Enrichment

**Overview:** Computes aggregate financial metrics from the extracted transactions and prepares human-readable summaries for reporting and logging.

**Nodes Involved:**
- Analyze Spending

#### Node: Analyze Spending
- **Type / Role:** `n8n-nodes-base.code` — transforms extracted statement JSON into summary metrics and formatted strings.
- **Configuration (interpreted):**
  - Reads: `const statement = $input.first().json.data;`
  - Computes:
    - `totalSpending`: sum of absolute values of negative amounts
    - `totalIncome`: sum of positive amounts
    - `categoryTotals`: totals per category for spending only (negative amounts)
    - `categoryBreakdown`: multi-line string sorted by highest spend, includes percent of total spending
    - `largeTransactions`: list of transactions where `abs(amount) > 500`
    - `savingsRate`: `((income - spending)/income)*100` (as string via `.toFixed(1)`), or `0` if income is 0
    - `transactionCount`, `processedAt` ISO timestamp
  - Returns a single item merging the original statement fields with computed fields.
- **Key variables/expressions:**
  - `$input.first().json.data`
  - `new Date().toISOString()`
- **Input / Output:**
  - **Input:** Output of PDF Vector node (expects `.json.data.transactions`)
  - **Output:** To **Log Monthly Summary**
- **Version-specific notes:** TypeVersion **2** (modern Code node).
- **Potential failures / edge cases:**
  - If PDF Vector output structure differs (e.g., data not in `$json.data`), the code throws.
  - If `transactions` is missing/empty, breakdown may be empty; division by zero handled indirectly only for savings rate (not for category percentage if totalSpending is 0). In the current code:
    - If `totalSpending === 0`, categoryBreakdown calculation would attempt `(amount/totalSpending)`; however `categoryTotals` would also be empty if there are no negative transactions, so `categoryBreakdown` becomes an empty string (no division occurs). If there is a negative transaction but totals sum to 0 (unlikely), percentage could error.
  - `largeTransactions` prints raw `tx.amount` without formatting and may show many decimals.
  - Savings rate is returned as a **string** (because `.toFixed(1)`), but Sheets mapping treats it as string anyway.

---

### Block 4 — Persistence (Google Sheets)

**Overview:** Writes one row per processed statement to a Google Sheet for historical tracking.

**Nodes Involved:**
- Log Monthly Summary

#### Node: Log Monthly Summary
- **Type / Role:** `n8n-nodes-base.googleSheets` — appends a row with period and summary metrics.
- **Configuration (interpreted):**
  - Operation: **append**
  - Document: Google Sheet named “Test-workflow” (ID: `18Jpu_lX1i9zkHo_TpS9lGYfvtRY7sHFo1qRbttj58YI`)
  - Sheet tab: “Trang tính1” (gid=0)
  - Mapping mode: **define below** with explicit columns:
    - Period Start: `={{ $json.statementPeriod.startDate }}`
    - Period End: `={{ $json.statementPeriod.endDate }}`
    - Opening Balance: `={{ $json.openingBalance }}`
    - Closing Balance: `={{ $json.closingBalance }}`
    - Total Income: `={{ $json.totalIncome }}`
    - Total Spending: `={{ $json.totalSpending }}`
    - Savings Rate %: `={{ $json.savingsRate }}`
    - Transaction Count: `={{ $json.transactionCount }}`
    - Processed Date: `={{ $json.processedAt.split('T')[0] }}`
  - Type conversion options appear disabled (`attemptToConvertTypes: false`, etc.), so values may be stored as strings depending on input types.
- **Key variables/expressions:**
  - `{{$json.processedAt.split('T')[0]}}` (extract date portion)
- **Input / Output:**
  - **Input:** From Analyze Spending enriched JSON
  - **Output:** To Send Analysis Report
- **Version-specific notes:** TypeVersion **4.4**.
- **Potential failures / edge cases:**
  - Missing sheet/tab or renamed columns can cause append failures.
  - OAuth permission issues (Sheets scope).
  - Rate limits if processing many statements rapidly.
  - If `statementPeriod` is missing, expressions resolve to empty or error depending on n8n expression handling.

---

### Block 5 — Reporting (Slack)

**Overview:** Posts a formatted monthly report to Slack including totals, category breakdown, and flagged large transactions.

**Nodes Involved:**
- Send Analysis Report

#### Node: Send Analysis Report
- **Type / Role:** `n8n-nodes-base.slack` — sends a message to a Slack channel using OAuth2.
- **Configuration (interpreted):**
  - Action: send message (via “text” parameter)
  - Target selection: **channel**
  - Channel ID: `C0A0JMC9VED` (cached name: `all-workflow`)
  - Message text uses Slack markdown and n8n expressions referencing the **Analyze Spending** node explicitly.
- **Key variables/expressions (examples):**
  - `{{ $('Analyze Spending').item.json.statementPeriod.startDate }}`
  - `{{ $('Analyze Spending').item.json.totalIncome.toFixed(2) }}`
  - `{{ $('Analyze Spending').item.json.categoryBreakdown }}`
  - `{{ $('Analyze Spending').item.json.largeTransactions }}`
- **Input / Output:**
  - **Input:** From Log Monthly Summary (but the message content references Analyze Spending directly)
  - **Output:** End of workflow
- **Version-specific notes:** TypeVersion **2.1**.
- **Potential failures / edge cases:**
  - If Slack OAuth scopes don’t include `chat:write` (and channel access), sending fails.
  - If Analyze Spending output is missing or the workflow changes node names, expression references break.
  - `.toFixed(2)` assumes totals are numbers; if they become strings, this throws.

---

### Sticky Note (Workflow description)
- **Type / Role:** `n8n-nodes-base.stickyNote` — documentation only.
- **Content (preserved):**
  - “🏦 Bank Statement Analyzer… Monitors Gmail… Extracts… Categorizes… Calculates savings rate… Identifies large transactions… Sends monthly summary… Perfect for: …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / overview | — | — | ## 🏦 Bank Statement Analyzer; **What this workflow does:** 1. Monitors Gmail for bank statements 2. Extracts all transactions 3. Categorizes spending 4. Calculates savings rate 5. Identifies large transactions 6. Sends monthly summary; **Perfect for:** Personal finance tracking; Small business bookkeeping; Budget analysis |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / category list | — | — | ## 📊 Spending Categories; Income; Utilities; Groceries; Dining; Shopping; Transport; Subscriptions; Healthcare; Entertainment; Transfers |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for new statement emails | — | Get a message |  |
| Get a message | n8n-nodes-base.gmail | Fetch full email + download attachments | Gmail Trigger | PDF Vector - Extract Statement |  |
| PDF Vector - Extract Statement | n8n-nodes-pdfvector.pdfVector | Extract statement + transactions from PDF attachment (schema constrained) | Get a message | Analyze Spending |  |
| Analyze Spending | n8n-nodes-base.code | Compute totals, savings rate, category breakdown, large tx list | PDF Vector - Extract Statement | Log Monthly Summary |  |
| Log Monthly Summary | n8n-nodes-base.googleSheets | Append monthly summary row to Google Sheets | Analyze Spending | Send Analysis Report |  |
| Send Analysis Report | n8n-nodes-base.slack | Post monthly report to Slack channel | Log Monthly Summary | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **“Bank Statement Analyzer & Budget Tracker”**
   - Set workflow to inactive while configuring (optional).

2. **Add Gmail Trigger**
   - Node type: **Gmail Trigger**
   - Credentials: **Gmail OAuth2**
     - Ensure scopes allow reading messages.
   - Configure:
     - Polling: **Every minute**
     - Include spam/trash: **Off**
   - (Recommended) Add filters (label/from/subject) to avoid triggering on all emails; the provided workflow does not include them.

3. **Add Gmail “Get a message” node**
   - Node type: **Gmail**
   - Operation: **Get**
   - Message ID: `={{ $json.id }}`
   - Options:
     - **Download Attachments: true**
     - Simple: **false**
   - Connect: **Gmail Trigger → Get a message**

4. **Add PDF Vector extraction node**
   - Node type: **PDF Vector**
   - Credentials: **PDF Vector API**
   - Configure:
     - Resource: **Document**
     - Operation: **Extract**
     - Input type: **File**
     - Binary property name: **`attachment_0`**
     - Prompt: include transaction extraction instructions and categorization rules (as in the workflow)
     - Schema: define an object with required `closingBalance` and `transactions[]`, and a `category` enum including:
       - Income, Utilities, Groceries, Dining, Shopping, Transport, Travel, Subscriptions, Healthcare, Entertainment, Transfer, Other
   - Connect: **Get a message → PDF Vector - Extract Statement**

5. **Add Code node (“Analyze Spending”)**
   - Node type: **Code**
   - Paste logic that:
     - Reads `const statement = $input.first().json.data;`
     - Computes `totalSpending`, `totalIncome`, `savingsRate`, `categoryTotals`, `categoryBreakdown`, `largeTransactions`, `transactionCount`, `processedAt`
     - Returns a single merged JSON item
   - Connect: **PDF Vector - Extract Statement → Analyze Spending**

6. **Add Google Sheets node (“Log Monthly Summary”)**
   - Node type: **Google Sheets**
   - Credentials: **Google Sheets OAuth2**
   - Operation: **Append**
   - Select the spreadsheet and sheet tab (create them if needed):
     - Spreadsheet: “Test-workflow” (or your own)
     - Sheet tab: “Trang tính1” (or your own)
   - Define columns and map:
     - Period Start → `={{ $json.statementPeriod.startDate }}`
     - Period End → `={{ $json.statementPeriod.endDate }}`
     - Opening Balance → `={{ $json.openingBalance }}`
     - Closing Balance → `={{ $json.closingBalance }}`
     - Total Income → `={{ $json.totalIncome }}`
     - Total Spending → `={{ $json.totalSpending }}`
     - Savings Rate % → `={{ $json.savingsRate }}`
     - Transaction Count → `={{ $json.transactionCount }}`
     - Processed Date → `={{ $json.processedAt.split('T')[0] }}`
   - Connect: **Analyze Spending → Log Monthly Summary**

7. **Add Slack node (“Send Analysis Report”)**
   - Node type: **Slack**
   - Credentials: **Slack OAuth2** (ensure `chat:write` and channel access)
   - Select: **Channel**
   - Channel: choose your target channel (the provided workflow uses `all-workflow`)
   - Text: compose a message and reference **Analyze Spending** results, for example:
     - Period, Income/Spending totals (formatted with `.toFixed(2)`)
     - Savings rate
     - Category breakdown string
     - Large transactions list
   - Connect: **Log Monthly Summary → Send Analysis Report**

8. **(Optional) Add Sticky Notes**
   - Add one note describing the workflow purpose and one listing category names, matching your schema.

9. **Test end-to-end**
   - Send a test email with a PDF statement as the **first attachment**.
   - Verify:
     - Gmail download created `attachment_0`
     - PDF Vector returns `json.data.transactions`
     - Sheets row is appended
     - Slack message posts correctly

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “🏦 Bank Statement Analyzer… Monitors Gmail… Extracts… Categorizes… Calculates savings rate… Identifies large transactions… Sends monthly summary…” | Sticky note (workflow overview) |
| “📊 Spending Categories: Income, Utilities, Groceries, Dining, Shopping, Transport, Subscriptions, Healthcare, Entertainment, Transfers” | Sticky note (category list). Note: schema also includes Travel, Other, and uses “Transfer” (singular). |