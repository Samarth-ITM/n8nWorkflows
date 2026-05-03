Reconcile Stripe payments and flag anomalies with Google Sheets and Gemini AI

https://n8nworkflows.xyz/workflows/reconcile-stripe-payments-and-flag-anomalies-with-google-sheets-and-gemini-ai-13980


# Reconcile Stripe payments and flag anomalies with Google Sheets and Gemini AI

## 1. Workflow Overview

This workflow performs a scheduled reconciliation between Stripe payment data and an accounting ledger stored in Google Sheets. It identifies discrepancies, optionally asks Gemini AI to analyze those anomalies, records the outcome in a reconciliation sheet, sends a Slack alert, and on Mondays sends a weekly email report.

The workflow is organized into the following logical blocks:

### 1.1 Scheduled Trigger and Runtime Context
The workflow starts on a schedule, then initializes user-defined settings that are meant to control downstream behavior such as date ranges, sheet names, thresholds, or account references.

### 1.2 Data Collection
Two parallel branches fetch the source datasets:
- Stripe transactions via HTTP request
- Accounting ledger rows from Google Sheets

These datasets converge into the reconciliation logic.

### 1.3 Reconciliation and Decisioning
A Code node compares Stripe transactions against ledger entries and determines whether discrepancies exist. An IF node then routes execution either toward anomaly handling or to a clean completion path.

### 1.4 AI-Based Discrepancy Analysis
If discrepancies are found, the workflow prepares a structured prompt payload and passes it to a LangChain LLM chain powered by the Google Gemini chat model.

### 1.5 Result Formatting and Downstream Notifications
The AI output is normalized in a Code node, then sent to:
- a Google Sheet for logging,
- Slack for alerting,
- and a weekday check for optional weekly email reporting.

### 1.6 Weekly Reporting
If the run occurs on Monday, the workflow sends a summary email using Gmail.

### 1.7 No-Discrepancy Completion Path
If no discrepancies are found, the workflow exits through a simple Set node used as a terminal marker.

---

## 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and Runtime Context

**Overview:**  
This block starts the workflow automatically and establishes the runtime parameters used by downstream nodes. It acts as the operational entry point and central configuration layer.

**Nodes Involved:**  
- Daily Reconciliation
- User Settings

### Node: Daily Reconciliation
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger that launches the workflow at configured intervals.
- **Configuration choices:**  
  No explicit schedule settings are present in the JSON, so this node is currently incomplete or relying on defaults from the editor state. In practice, it should be configured for a daily run.
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - Input: none
  - Output: User Settings
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Workflow never runs if no schedule is configured
  - Unexpected timezone behavior if timezone is not explicitly defined in workflow settings
  - Duplicate runs if multiple schedules are later added
- **Sub-workflow reference:**  
  None

### Node: User Settings
- **Type and technical role:** `n8n-nodes-base.set`  
  Intended to define reusable workflow variables such as date windows, Stripe filters, ledger sheet metadata, or notification settings.
- **Configuration choices:**  
  The node has no visible parameters in the JSON, so it is currently empty. In a functioning implementation, this is where constants should be declared.
- **Key expressions or variables used:**  
  None shown, but likely intended outputs could include values like:
  - reconciliation date
  - Stripe API endpoint
  - Google Sheet document ID
  - discrepancy threshold
- **Input and output connections:**  
  - Input: Daily Reconciliation
  - Outputs: Fetch Stripe Transactions, Read Accounting Ledger
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Downstream expressions may fail if expected fields are not defined here
  - Empty configuration makes the workflow hard to parameterize and maintain
- **Sub-workflow reference:**  
  None

---

## 2.2 Data Collection

**Overview:**  
This block retrieves the two datasets that will be reconciled: Stripe payments and accounting ledger entries. Both branches feed into the reconciliation engine.

**Nodes Involved:**  
- Fetch Stripe Transactions
- Read Accounting Ledger

### Node: Fetch Stripe Transactions
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Makes a direct API call to Stripe to retrieve payment-related data.
- **Configuration choices:**  
  The node parameters are empty in the JSON, but its note indicates it should use HTTP Header Auth with:
  - Header name: `Authorization`
  - Value: `Bearer sk_live_YOUR_KEY`
  
  A production-safe configuration would call a Stripe endpoint such as `/v1/payment_intents`, `/v1/charges`, or `/v1/balance_transactions` depending on reconciliation needs.
- **Key expressions or variables used:**  
  None visible. Expected expressions may reference `User Settings` for date filters or pagination.
- **Input and output connections:**  
  - Input: User Settings
  - Output: Reconciliation Logic
- **Version-specific requirements:**  
  Type version `4.2`
- **Edge cases or potential failure types:**  
  - 401/403 authentication failures due to invalid Stripe key
  - Wrong endpoint choice leading to incomplete reconciliation data
  - Pagination not handled, resulting in missing transactions
  - Rate limiting
  - Amount formatting mismatches, especially if Stripe values are in minor units
  - Currency mismatches if multiple currencies exist
- **Sub-workflow reference:**  
  None

### Node: Read Accounting Ledger
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads accounting records from a Google Sheet used as the internal ledger.
- **Configuration choices:**  
  Parameters are empty in the JSON. In practice, it should be configured to read rows from a specific spreadsheet and sheet/tab, likely filtered by date or status.
- **Key expressions or variables used:**  
  None visible. Likely candidates:
  - Spreadsheet ID from User Settings
  - Sheet/tab name
  - Date-range expressions
- **Input and output connections:**  
  - Input: User Settings
  - Output: Reconciliation Logic
- **Version-specific requirements:**  
  Type version `4.5`
- **Edge cases or potential failure types:**  
  - Google authentication issues
  - Wrong sheet or range selection
  - Header row problems causing field names to shift
  - Inconsistent ledger formatting, such as text vs numeric amounts
  - Empty result set for the selected period
- **Sub-workflow reference:**  
  None

---

## 2.3 Reconciliation and Decisioning

**Overview:**  
This block compares Stripe and ledger records, produces discrepancy findings, and determines whether the workflow should proceed with AI-assisted analysis or stop cleanly.

**Nodes Involved:**  
- Reconciliation Logic
- Has Discrepancies?
- Workflow Complete

### Node: Reconciliation Logic
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript reconciliation engine that merges inputs from Stripe and Google Sheets and identifies missing, mismatched, or duplicate records.
- **Configuration choices:**  
  The node has no visible code in the JSON, but it is clearly intended to perform:
  - transaction matching
  - discrepancy detection
  - structured output generation for the IF node and AI prompt
- **Key expressions or variables used:**  
  Not visible. Likely patterns in a working version would include:
  - accessing both incoming input streams
  - normalizing dates, IDs, amounts, currencies
  - computing discrepancy counts
- **Input and output connections:**  
  - Inputs: Fetch Stripe Transactions, Read Accounting Ledger
  - Output: Has Discrepancies?
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Multi-input handling mistakes if the code assumes a single input stream
  - Amount comparison issues due to cents vs decimal representation
  - Duplicate transaction IDs
  - Timezone-related date mismatches
  - Missing keys in either dataset
  - Runtime exceptions from undefined fields
- **Sub-workflow reference:**  
  None

### Node: Has Discrepancies?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router deciding whether discrepancies exist.
- **Configuration choices:**  
  No explicit condition is present in the JSON. It should evaluate a field produced by Reconciliation Logic, such as:
  - discrepancy count > 0
  - anomalies array is not empty
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input: Reconciliation Logic
  - True output: Prepare AI Analysis
  - False output: Workflow Complete
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Condition referencing a missing field
  - Truthy/falsy confusion if counts are strings
  - Empty arrays mishandled
- **Sub-workflow reference:**  
  None

### Node: Workflow Complete
- **Type and technical role:** `n8n-nodes-base.set`  
  Terminal node used to mark successful completion when no discrepancies are found.
- **Configuration choices:**  
  No fields are defined. In a better implementation, this could set a status like `no_discrepancies = true`.
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - Input: Has Discrepancies? (false branch)
  - Output: none
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  Minimal; mainly useful as a placeholder end state.
- **Sub-workflow reference:**  
  None

---

## 2.4 AI-Based Discrepancy Analysis

**Overview:**  
This block packages discrepancy details for LLM consumption, runs an AI analysis, and relies on Gemini as the language model provider. It is only activated when anomalies are detected.

**Nodes Involved:**  
- Prepare AI Analysis
- AI Discrepancy Analysis
- Gemini AI Model

### Node: Prepare AI Analysis
- **Type and technical role:** `n8n-nodes-base.set`  
  Structures the discrepancy payload into fields that can be passed into the LangChain node as prompt variables or message content.
- **Configuration choices:**  
  No parameters are shown, but it would typically define:
  - discrepancy summary
  - list of unmatched records
  - instructions for root-cause analysis
  - severity or business context
- **Key expressions or variables used:**  
  Not visible. Likely references fields from Reconciliation Logic.
- **Input and output connections:**  
  - Input: Has Discrepancies? (true branch)
  - Output: AI Discrepancy Analysis
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Large discrepancy arrays causing oversized prompts
  - Missing fields from the reconciliation output
  - Poor prompt structuring leading to weak AI output
- **Sub-workflow reference:**  
  None

### Node: AI Discrepancy Analysis
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Executes an LLM chain using prepared input and a connected language model.
- **Configuration choices:**  
  No prompt template or chain settings are visible. In a working setup, this node should include:
  - an instruction prompt
  - expected output style, ideally structured JSON or Markdown
  - references to discrepancy fields from Prepare AI Analysis
- **Key expressions or variables used:**  
  None visible, but likely prompt variables populated from incoming JSON.
- **Input and output connections:**  
  - Main input: Prepare AI Analysis
  - AI language model input: Gemini AI Model
  - Main output: Format Analysis Results
- **Version-specific requirements:**  
  Type version `1.4`  
  Requires n8n with LangChain-compatible AI nodes installed/enabled.
- **Edge cases or potential failure types:**  
  - Prompt variable mismatch
  - LLM timeout
  - Quota exhaustion
  - Non-deterministic output format
  - Hallucinated causes if raw data is incomplete
- **Sub-workflow reference:**  
  None

### Node: Gemini AI Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Supplies the Google Gemini chat model to the LangChain chain node.
- **Configuration choices:**  
  No model settings are shown. In practice, this should include:
  - Google AI / Gemini credentials
  - model selection
  - optional temperature and token limits
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - Output via `ai_languageModel` connection to AI Discrepancy Analysis
- **Version-specific requirements:**  
  Type version `1`  
  Requires AI-capable n8n version and valid Gemini credentials.
- **Edge cases or potential failure types:**  
  - Invalid API key or auth configuration
  - Model not available in region/account
  - Token/context limits exceeded
  - Safety filters blocking analysis phrasing
- **Sub-workflow reference:**  
  None

---

## 2.5 Result Formatting and Downstream Notifications

**Overview:**  
This block converts the AI output into a loggable and actionable format, then distributes it to storage and messaging destinations.

**Nodes Involved:**  
- Format Analysis Results
- Log to Reconciliation Sheet
- Send Slack Alert
- Is Monday?

### Node: Format Analysis Results
- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes the LLM output and creates destination-specific fields for Sheets, Slack, and reporting.
- **Configuration choices:**  
  No code is visible. A practical implementation would:
  - parse AI output
  - set summary text
  - set severity/status
  - prepare row values for Sheets
  - prepare Slack message body
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input: AI Discrepancy Analysis
  - Outputs: Log to Reconciliation Sheet, Send Slack Alert, Is Monday?
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - JSON parsing failure if the AI output is unstructured
  - Missing expected response fields
  - Message truncation for Slack
  - Sheet schema mismatch
- **Sub-workflow reference:**  
  None

### Node: Log to Reconciliation Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Writes reconciliation results and AI findings into a Google Sheet for record-keeping.
- **Configuration choices:**  
  Parameters are empty in the JSON. It should be configured to append a row or update a reporting sheet.
- **Key expressions or variables used:**  
  None visible. Likely expected:
  - timestamp
  - discrepancy count
  - anomaly summary
  - AI analysis
- **Input and output connections:**  
  - Input: Format Analysis Results
  - Output: none
- **Version-specific requirements:**  
  Type version `4.5`
- **Edge cases or potential failure types:**  
  - Invalid spreadsheet access
  - Column mapping mismatch
  - Append vs update mode confusion
  - Large text fields exceeding practical usability
- **Sub-workflow reference:**  
  None

### Node: Send Slack Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends an alert about reconciliation discrepancies to Slack.
- **Configuration choices:**  
  No channel or message settings are shown in the JSON. In practice, configure:
  - Slack credentials
  - channel or conversation ID
  - message text from formatted results
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input: Format Analysis Results
  - Output: none
- **Version-specific requirements:**  
  Type version `2.2`
- **Edge cases or potential failure types:**  
  - Invalid Slack token or revoked app permissions
  - Missing target channel
  - Message formatting issues
  - Rate limits
- **Sub-workflow reference:**  
  None

### Node: Is Monday?
- **Type and technical role:** `n8n-nodes-base.if`  
  Determines whether to send a weekly report email.
- **Configuration choices:**  
  No condition is shown. It should likely test the current date’s weekday, for example from `$now` or a formatted date generated earlier.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input: Format Analysis Results
  - True output: Send Weekly Email Report
  - False output: none
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Locale/timezone mismatch causing wrong weekday
  - Incorrect weekday indexing
  - Condition evaluates against non-date string
- **Sub-workflow reference:**  
  None

---

## 2.6 Weekly Reporting

**Overview:**  
This block sends a Monday-only email report summarizing reconciliation findings. It is a secondary notification path, likely aimed at finance or operations stakeholders.

**Nodes Involved:**  
- Send Weekly Email Report

### Node: Send Weekly Email Report
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email through Gmail when the weekly condition is met.
- **Configuration choices:**  
  Parameters are empty in the JSON. It should be configured with:
  - recipient(s)
  - subject
  - body containing the weekly discrepancy summary
- **Key expressions or variables used:**  
  None visible. Likely references output from Format Analysis Results.
- **Input and output connections:**  
  - Input: Is Monday?
  - Output: none
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - Gmail OAuth misconfiguration
  - Invalid recipient list
  - HTML/text formatting issues
  - Sending limits or account restrictions
- **Sub-workflow reference:**  
  None

---

## 2.7 Sticky Notes and Documentation Layer

**Overview:**  
The workflow contains four sticky note nodes used as visual annotations, but their content is empty. One operational note exists directly on the Stripe HTTP node.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas annotation only; no execution role.
- **Configuration choices:**  
  Empty content.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas annotation only.
- **Configuration choices:**  
  Empty content.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas annotation only.
- **Configuration choices:**  
  Empty content.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas annotation only.
- **Configuration choices:**  
  Empty content.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Visual annotation |  |  |  |
| Sticky Note1 | Sticky Note | Visual annotation |  |  |  |
| Sticky Note2 | Sticky Note | Visual annotation |  |  |  |
| Sticky Note3 | Sticky Note | Visual annotation |  |  |  |
| Daily Reconciliation | Schedule Trigger | Starts the workflow on a schedule |  | User Settings |  |
| User Settings | Set | Defines runtime configuration values | Daily Reconciliation | Fetch Stripe Transactions; Read Accounting Ledger |  |
| Fetch Stripe Transactions | HTTP Request | Retrieves Stripe payment data | User Settings | Reconciliation Logic | Set up HTTP Header Auth with Name: Authorization, Value: Bearer sk_live_YOUR_KEY |
| Read Accounting Ledger | Google Sheets | Reads internal ledger data | User Settings | Reconciliation Logic |  |
| Reconciliation Logic | Code | Compares Stripe and ledger records | Fetch Stripe Transactions; Read Accounting Ledger | Has Discrepancies? |  |
| Has Discrepancies? | If | Routes based on anomaly presence | Reconciliation Logic | Prepare AI Analysis; Workflow Complete |  |
| Prepare AI Analysis | Set | Structures prompt input for AI | Has Discrepancies? | AI Discrepancy Analysis |  |
| AI Discrepancy Analysis | LangChain LLM Chain | Generates AI interpretation of discrepancies | Prepare AI Analysis; Gemini AI Model | Format Analysis Results |  |
| Gemini AI Model | Google Gemini Chat Model | Provides the LLM backend |  | AI Discrepancy Analysis |  |
| Format Analysis Results | Code | Normalizes AI output for logging and notifications | AI Discrepancy Analysis | Log to Reconciliation Sheet; Send Slack Alert; Is Monday? |  |
| Log to Reconciliation Sheet | Google Sheets | Logs reconciliation results | Format Analysis Results |  |  |
| Send Slack Alert | Slack | Sends discrepancy alert to Slack | Format Analysis Results |  |  |
| Is Monday? | If | Checks whether weekly email should be sent | Format Analysis Results | Send Weekly Email Report |  |
| Send Weekly Email Report | Gmail | Sends Monday summary email | Is Monday? |  |  |
| Workflow Complete | Set | Marks successful end when no discrepancy exists | Has Discrepancies? |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **Reconcile Stripe payments and flag anomalies with Google Sheets and Gemini AI**

2. **Add a Schedule Trigger node**
   - Type: `Schedule Trigger`
   - Rename it to: **Daily Reconciliation**
   - Configure it to run once per day
   - Set the preferred timezone in workflow settings if finance reporting depends on a business timezone

3. **Add a Set node**
   - Type: `Set`
   - Rename it to: **User Settings**
   - Connect **Daily Reconciliation → User Settings**
   - Add fields that will be reused later, for example:
     - `stripeBaseUrl`
     - `stripeEndpoint`
     - `reconciliationDate`
     - `ledgerSpreadsheetId`
     - `ledgerSheetName`
     - `logSpreadsheetId`
     - `logSheetName`
     - `slackChannel`
     - `weeklyEmailRecipients`
   - If you do not want constants hardcoded elsewhere, define them here

4. **Add an HTTP Request node**
   - Type: `HTTP Request`
   - Rename it to: **Fetch Stripe Transactions**
   - Connect **User Settings → Fetch Stripe Transactions**
   - Configure:
     - Method: usually `GET`
     - URL: Stripe endpoint appropriate for your reconciliation model
     - Authentication: use HTTP Header Auth
       - Header name: `Authorization`
       - Value: `Bearer <your_stripe_secret_key>`
   - Recommended:
     - add query parameters for date range
     - enable pagination if pulling multiple records
   - Add the node note:
     - `Set up HTTP Header Auth with Name: Authorization, Value: Bearer sk_live_YOUR_KEY`

5. **Add a Google Sheets node**
   - Type: `Google Sheets`
   - Rename it to: **Read Accounting Ledger**
   - Connect **User Settings → Read Accounting Ledger**
   - Configure Google credentials
   - Set operation to read rows from your accounting ledger sheet
   - Define:
     - Spreadsheet ID
     - Sheet/tab name
     - Header row usage
   - If relevant, filter records by date or reconciliation period

6. **Add a Code node**
   - Type: `Code`
   - Rename it to: **Reconciliation Logic**
   - Connect:
     - **Fetch Stripe Transactions → Reconciliation Logic**
     - **Read Accounting Ledger → Reconciliation Logic**
   - In the code, implement logic to:
     - gather both inputs
     - normalize Stripe amounts and ledger amounts
     - compare by transaction ID, payment reference, amount, currency, and date
     - produce fields such as:
       - `hasDiscrepancies`
       - `discrepancyCount`
       - `discrepancies`
       - `matchedCount`
       - `missingInLedger`
       - `missingInStripe`
       - `amountMismatches`
   - Ensure output is a predictable JSON structure

7. **Add an IF node**
   - Type: `If`
   - Rename it to: **Has Discrepancies?**
   - Connect **Reconciliation Logic → Has Discrepancies?**
   - Configure condition based on reconciliation output, for example:
     - `hasDiscrepancies` equals `true`
     - or `discrepancyCount` greater than `0`

8. **Add a Set node for the true branch**
   - Type: `Set`
   - Rename it to: **Prepare AI Analysis**
   - Connect the **true** output of **Has Discrepancies? → Prepare AI Analysis**
   - Add fields to construct a prompt payload, for example:
     - `summary`
     - `discrepancyList`
     - `analysisInstruction`
     - `businessContext`
   - Include a concise instruction asking the AI to identify likely causes and risk level

9. **Add a Code node or terminal marker for the false branch**
   - Type: `Set`
   - Rename it to: **Workflow Complete**
   - Connect the **false** output of **Has Discrepancies? → Workflow Complete**
   - Optionally add fields like:
     - `status = "complete"`
     - `hasDiscrepancies = false`

10. **Add a Gemini model node**
    - Type: `Google Gemini Chat Model`
    - Rename it to: **Gemini AI Model**
    - Configure Gemini credentials
    - Select the desired model
    - Optionally set:
      - temperature
      - max output tokens

11. **Add a LangChain LLM Chain node**
    - Type: `LLM Chain`
    - Rename it to: **AI Discrepancy Analysis**
    - Connect:
      - **Prepare AI Analysis → AI Discrepancy Analysis**
      - **Gemini AI Model → AI Discrepancy Analysis** using the language-model connection
    - Configure the prompt template using fields from **Prepare AI Analysis**
    - Strongly prefer structured output instructions such as:
      - summary
      - likely causes
      - severity
      - recommended next actions

12. **Add a Code node**
    - Type: `Code`
    - Rename it to: **Format Analysis Results**
    - Connect **AI Discrepancy Analysis → Format Analysis Results**
    - Implement formatting logic to produce:
      - a row payload for Google Sheets
      - a Slack message string
      - an email summary body
      - timestamp and status fields
    - If the AI output is structured JSON, parse and validate it here

13. **Add a Google Sheets node for logging**
    - Type: `Google Sheets`
    - Rename it to: **Log to Reconciliation Sheet**
    - Connect **Format Analysis Results → Log to Reconciliation Sheet**
    - Configure:
      - Spreadsheet ID for reconciliation log
      - Sheet/tab name
      - append-row or update-row mode
    - Map fields like:
      - timestamp
      - discrepancy count
      - AI summary
      - severity
      - recommended actions

14. **Add a Slack node**
    - Type: `Slack`
    - Rename it to: **Send Slack Alert**
    - Connect **Format Analysis Results → Send Slack Alert**
    - Configure Slack credentials
    - Select operation to send a message
    - Set channel and message body using formatted fields

15. **Add an IF node for weekday routing**
    - Type: `If`
    - Rename it to: **Is Monday?**
    - Connect **Format Analysis Results → Is Monday?**
    - Configure a condition checking whether the current weekday is Monday
    - Use a date expression based on workflow timezone to avoid off-by-one-day errors

16. **Add a Gmail node**
    - Type: `Gmail`
    - Rename it to: **Send Weekly Email Report**
    - Connect the **true** output of **Is Monday? → Send Weekly Email Report**
    - Configure Gmail OAuth2 credentials
    - Set:
      - recipients
      - subject
      - body from the formatted weekly summary
    - If desired, send HTML email for easier reading

17. **Optionally add sticky notes**
    - Add four sticky notes if you want the canvas to mirror the original layout
    - The provided workflow includes four empty sticky notes, so they are not functionally required

18. **Verify all credentials**
    - Stripe: secret API key via HTTP header auth
    - Google Sheets: Google OAuth2 or service-account-compatible method supported by your n8n setup
    - Gemini: Google AI / Gemini credentials
    - Slack: Slack app token or OAuth connection
    - Gmail: Gmail OAuth2

19. **Test each section independently**
    - Trigger manually
    - Confirm Stripe data returns expected fields
    - Confirm Google Sheets ledger rows are parsed correctly
    - Validate reconciliation logic with sample matches and mismatches
    - Ensure AI output is stable enough for downstream parsing
    - Confirm Slack and Gmail messages render correctly

20. **Harden the implementation**
    - Add pagination for Stripe
    - Add error handling or error branches
    - Normalize amounts and currencies explicitly
    - Limit AI prompt size for large discrepancy sets
    - Use structured AI output to reduce parsing failure
    - Add retry or alerting strategy for failed API calls

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Set up HTTP Header Auth with Name: Authorization, Value: Bearer sk_live_YOUR_KEY | Attached as a node note on Fetch Stripe Transactions |
| Four sticky note nodes are present but contain no text | Canvas documentation layer |
| The workflow JSON does not include actual node parameter values for schedule, API endpoints, prompts, conditions, message templates, or code bodies | Important implementation gap when rebuilding |
| No sub-workflows are used in this workflow | Architecture note |

### Additional implementation observations
- The workflow has **one entry point**: **Daily Reconciliation**.
- It does **not** contain any sub-workflow invocation nodes.
- Several nodes are structural placeholders with empty configurations in the provided JSON. The workflow architecture is clear, but the operational details must be filled in manually before it can run successfully.
- The most critical missing pieces are:
  - Stripe endpoint and pagination strategy
  - Google Sheets read/write mapping
  - reconciliation code
  - IF conditions
  - AI prompt design
  - Slack and Gmail message content