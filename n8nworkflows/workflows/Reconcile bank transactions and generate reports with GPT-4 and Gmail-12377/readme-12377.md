Reconcile bank transactions and generate reports with GPT-4 and Gmail

https://n8nworkflows.xyz/workflows/reconcile-bank-transactions-and-generate-reports-with-gpt-4-and-gmail-12377


# Reconcile bank transactions and generate reports with GPT-4 and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Reconcile bank transactions and generate reports with GPT-4 and Gmail  
**Workflow name (in JSON):** AI-Powered Financial Transaction Reconciliation & Reporting System

---

## 1. Workflow Overview

This workflow automates a daily accounting cycle: it pulls bank transactions and accounting-system records, uses multiple GPT-4o-powered agent nodes (LangChain in n8n) to classify and reconcile transactions, generates journal entries, performs error checks, builds financial statements and tax reports, communicates with a tax agent via Gmail, posts journal entries back to an accounting API, and emails a final daily summary.

### 1.1 Scheduling & Configuration
Runs every day at a fixed hour and centralizes environment parameters (API endpoints, emails, company metadata).

### 1.2 Data Ingestion
Fetches raw transaction data from a bank API and reference/ledger data from an accounting system API.

### 1.3 AI Classification & Reconciliation
Classifies transactions to accounts/categories, then matches bank vs system records and identifies discrepancies.

### 1.4 Journalization & Validation
Generates double-entry journal entries, detects posting/math issues, and (independently) posts journal entries to an external system.

### 1.5 Statements, Tax, Communication, and Reporting
Creates financial statements, computes tax reporting outputs, drafts/sends a tax-agent email via Gmail tool, aggregates results, then emails a daily executive summary.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Workflow Configuration
**Overview:** Triggers the workflow daily and sets shared configuration variables (API URLs, recipient emails, company details) used throughout the workflow via expressions.

**Nodes involved:**
- Daily Accounting Run
- Workflow Configuration

#### Node: Daily Accounting Run
- **Type / role:** `Schedule Trigger` — entry point, time-based execution.
- **Config:** Runs daily at **02:00** (triggerAtHour = 2).
- **Outputs:** Main → Workflow Configuration.
- **Failure/edge cases:** n8n timezone settings can shift run time; ensure instance timezone is correct.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central parameter store.
- **Config choices:** Adds fields while **including other fields**.
- **Key fields set (placeholders):**
  - `bankApiUrl`, `accountingSystemApiUrl`, `journalEntryApiUrl`
  - `taxAgentEmail`, `summaryRecipientEmail`
  - `companyName`, `fiscalYearEnd`
- **Expressions used by others:** `$('Workflow Configuration').first().json.<field>`
- **Outputs:** Main → Fetch Bank Transactions, Fetch Accounting System Data.
- **Failure/edge cases:** Placeholder values must be replaced; missing/invalid URLs will break HTTP nodes.

---

### Block 2 — Financial Data Ingestion & Combination
**Overview:** Pulls data from two external systems (bank + accounting) and merges them into one combined payload for downstream AI processing.

**Nodes involved:**
- Fetch Bank Transactions
- Fetch Accounting System Data
- Combine Financial Data

#### Node: Fetch Bank Transactions
- **Type / role:** `HTTP Request` — fetches bank transactions.
- **Config:**
  - URL: `={{ $('Workflow Configuration').first().json.bankApiUrl }}`
  - Sends header `Content-Type: application/json`
  - Auth: `predefinedCredentialType` (credential type not specified in JSON; must be set in n8n)
- **Input:** Workflow Configuration
- **Output:** Main → Combine Financial Data (input 0)
- **Failure/edge cases:** auth/403, wrong endpoint, pagination not handled, unexpected response shape (array vs object).

#### Node: Fetch Accounting System Data
- **Type / role:** `HTTP Request` — fetches accounting-system records (ledger/transactions).
- **Config:**
  - URL: `={{ $('Workflow Configuration').first().json.accountingSystemApiUrl }}`
  - Header `Content-Type: application/json`
  - Auth: `predefinedCredentialType`
- **Input:** Workflow Configuration
- **Output:** Main → Combine Financial Data (input 1)
- **Failure/edge cases:** same as above; also potential rate limits/timeouts.

#### Node: Combine Financial Data
- **Type / role:** `Merge` — combines data streams.
- **Config:** Mode `combine`, `combineBy: combineAll` (combines all items/data together).
- **Inputs:** Fetch Bank Transactions (0), Fetch Accounting System Data (1)
- **Output:** Main → Transaction Classifier Agent
- **Failure/edge cases:** If one side returns 0 items, merge behavior may produce empty or partial combined data depending on n8n merge semantics and upstream item counts.

---

### Block 3 — Transaction Classification (AI Agent + Structured Output)
**Overview:** Uses a LangChain Agent powered by GPT-4o to categorize each transaction and produce structured classification output via a schema parser.

**Nodes involved:**
- Transaction Classifier Agent
- OpenAI GPT (language model for the agent)
- Transaction Classification Schema
- Calculator Tool (available as an agent tool)

#### Node: Transaction Classifier Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent to classify transactions.
- **Config:**
  - Input text: `={{ $json }}` (passes the full merged JSON to the agent)
  - System message: detailed accounting classification instructions (category, account codes, flags suspicious items)
  - `promptType: define`
  - `hasOutputParser: true` (paired with Transaction Classification Schema)
  - Has access to **Calculator Tool**
- **Language model:** OpenAI GPT (connected via `ai_languageModel`)
- **Input:** Combine Financial Data
- **Output:** Main → Account Reconciliation Agent
- **Failure/edge cases:**
  - Token limits if bank+system payload is large (consider summarizing or batching per transaction).
  - Model may return data not matching schema; parser will fail.

#### Node: OpenAI GPT
- **Type / role:** `lmChatOpenAi` — provides GPT-4o for Transaction Classifier Agent.
- **Config:** Model `gpt-4o`, temperature `0.2`.
- **Credentials:** OpenAI API credential “OpenAi account”.
- **Used by:** Transaction Classifier Agent only.
- **Failure/edge cases:** invalid API key, model access issues, rate limits.

#### Node: Transaction Classification Schema
- **Type / role:** `outputParserStructured` — validates/structures agent output.
- **Schema (manual):** object with fields like `transactionId`, `category`, `accountCode`, `accountName`, `transactionType`, `amount`, `flags[]`.
- **Connected to:** Transaction Classifier Agent via `ai_outputParser`.
- **Failure/edge cases:** If agent returns an array for multiple transactions but schema expects a single object, parsing will fail (schema design should match expected multiplicity).

#### Node: Calculator Tool
- **Type / role:** `toolCalculator` — arithmetic tool usable by multiple agents.
- **Connected as tool to:** Transaction Classifier Agent, Account Reconciliation Agent, Journal Entry Generator Agent, Error Detection Agent, Financial Statement Generator Agent, Tax Report Generator Agent.
- **Failure/edge cases:** Typically safe; tool-use depends on agent deciding to call it.

---

### Block 4 — Reconciliation & Matching (AI Agent + Structured Output)
**Overview:** Compares bank and accounting system records, identifies matches/unmatched items, and returns discrepancy analysis and actions.

**Nodes involved:**
- Account Reconciliation Agent
- OpenAI (language model for the agent)
- Reconciliation Schema
- Calculator Tool (available as an agent tool)

#### Node: Account Reconciliation Agent
- **Type / role:** LangChain Agent — performs matching and discrepancy analysis.
- **Config:**
  - Input text: `={{ $json }}`
  - System message: matching, discrepancies, suggested corrective actions
  - Output parser enabled (schema-based)
- **Language model:** OpenAI (lmChatOpenAi node named “OpenAI”)
- **Input:** Transaction Classifier Agent
- **Output:** Main → Journal Entry Generator Agent
- **Failure/edge cases:** Without explicit transaction identifiers in the combined dataset, matching quality may be poor; schema mismatch will fail parse.

#### Node: OpenAI
- **Type / role:** `lmChatOpenAi` — provides GPT-4o for Account Reconciliation Agent.
- **Config:** Model `gpt-4o`, temperature `0.2`.
- **Credentials:** “OpenAi account”.
- **Failure/edge cases:** same as other OpenAI nodes.

#### Node: Reconciliation Schema
- **Type / role:** `outputParserStructured` — structured reconciliation output.
- **Config:** Uses a JSON example (matchedItems, unmatchedBankItems, unmatchedSystemItems, discrepancies, totalDifference, recommendedActions).
- **Connected to:** Account Reconciliation Agent via `ai_outputParser`.
- **Failure/edge cases:** Example implies arrays; ensure agent is instructed to always return the same top-level object structure.

---

### Block 5 — Journal Entry Generation & Posting
**Overview:** Produces double-entry journals from reconciliation results, validates them for issues, and posts them to an external journal-entry API.

**Nodes involved:**
- Journal Entry Generator Agent
- OpenAI GPT-6 (language model for the agent)
- Journal Entry Schema
- Error Detection Agent
- Error Detection Schema
- Post Journal Entries to System
- Calculator Tool

#### Node: Journal Entry Generator Agent
- **Type / role:** LangChain Agent — generates journal entries (debits/credits).
- **Config:**
  - Input text: `={{ $json }}`
  - System message: GAAP/IFRS compliant double-entry, debits=credits, posting date, references
  - Output parser enabled (Journal Entry Schema)
- **Language model:** OpenAI GPT-6
- **Input:** Account Reconciliation Agent
- **Outputs:**
  - Main → Error Detection Agent
  - Main → Post Journal Entries to System
- **Failure/edge cases:** If multiple transactions are provided but schema expects one entry object, outputs may not parse; large batches may exceed context.

#### Node: OpenAI GPT-6
- **Type / role:** `lmChatOpenAi` — GPT-4o for Journal Entry Generator Agent.
- **Config:** Model `gpt-4o`, temperature `0.2`.
- **Credentials:** “OpenAi account”.

#### Node: Journal Entry Schema
- **Type / role:** `outputParserStructured` — enforces journal entry structure.
- **Schema:** entryNumber, date, entries[] (accountCode/accountName/description/debit/credit), totalDebit, totalCredit, reference.
- **Connected to:** Journal Entry Generator Agent via `ai_outputParser`.
- **Edge cases:** Totals might be inconsistent; schema won’t enforce debit=credit equality unless the agent computes correctly (consider adding post-validation).

#### Node: Error Detection Agent
- **Type / role:** LangChain Agent — detects accounting/math/duplication errors in journal output.
- **Config:**
  - Input text: `={{ $json }}`
  - System message: checks equality, duplicates, wrong accounts, unusual patterns; assigns severity
  - Output parser enabled (Error Detection Schema)
- **Language model:** (Not explicitly connected to a dedicated LLM node in connections; it *does* have Calculator Tool attached. In practice, you should connect an `lmChatOpenAi` node as `ai_languageModel` to this agent—currently missing in the connections list.)
- **Input:** Journal Entry Generator Agent
- **Output:** Main → Check for Errors
- **Failure/edge cases:** If no LLM is attached, node will fail at runtime. If attached, parser may fail if agent returns free text.

#### Node: Error Detection Schema
- **Type / role:** `outputParserStructured`
- **Schema:** `{ errors: [{errorType, description, affectedEntries[], severity, recommendedCorrection}], errorCount, criticalErrorCount }`
- **Connected to:** Error Detection Agent via `ai_outputParser`.

#### Node: Post Journal Entries to System
- **Type / role:** `HTTP Request` — posts generated journal entries to an external system.
- **Config:**
  - URL: `={{ $('Workflow Configuration').first().json.journalEntryApiUrl }}`
  - Method: `POST`
  - Body: JSON `={{ $json }}`
  - Header: `Content-Type: application/json`
  - Auth: `predefinedCredentialType`
- **Input:** Journal Entry Generator Agent
- **Output:** Main → Aggregate All Results
- **Failure/edge cases:** downstream system validation errors (400), auth/401, idempotency (duplicates if rerun), schema mismatch with target API.

---

### Block 6 — Error Gate → Statements → Tax → Tax-Agent Email
**Overview:** If errors exist, workflow still continues (as built) into financial statement generation; then tax report generation; then an AI agent uses Gmail Tool to email the tax agent.

**Nodes involved:**
- Check for Errors
- Financial Statement Generator Agent
- OpenAI GPT-5
- Financial Statement Schema
- Tax Report Generator Agent
- OpenAI GPT-
- Tax Report Schema
- Tax Agent Communication Agent
- OpenAI GPT-4
- Gmail Tool
- Calculator Tool

#### Node: Check for Errors
- **Type / role:** `If` — checks whether error array exists and length > 0.
- **Conditions:**
  - `$('Error Detection Agent').item.json.errors` is not empty
  - length of errors > 0
- **Output:** Only **true** path is connected → Financial Statement Generator Agent.
- **Edge cases / logic note:** If there are **no errors**, the workflow has **no connected false branch**, so statement/tax/reporting may never run. If the intent is “continue regardless,” connect both outputs or invert logic.

#### Node: Financial Statement Generator Agent
- **Type / role:** LangChain Agent — builds Balance Sheet / Income Statement / Cash Flow + ratios.
- **Config:** Input `={{ $json }}`, structured output enabled.
- **Language model:** OpenAI GPT-5
- **Input:** Check for Errors (true path)
- **Output:** Main → Tax Report Generator Agent
- **Failure/edge cases:** needs enough structured journal data; otherwise hallucinated statements.

#### Node: OpenAI GPT-5
- **Type / role:** `lmChatOpenAi` for Financial Statement Generator Agent.
- **Config:** Model `gpt-4o`, temperature `0.2`.

#### Node: Financial Statement Schema
- **Type / role:** `outputParserStructured`
- **Schema:** nested objects for balanceSheet (assets/liabilities/equity), incomeStatement (revenue/expenses/netIncome), cashFlowStatement (operating/investing/financing), financialRatios (currentRatio, debtToEquity, profitMargin).
- **Connected to:** Financial Statement Generator Agent via `ai_outputParser`.

#### Node: Tax Report Generator Agent
- **Type / role:** LangChain Agent — computes taxable income, deductions, liabilities, schedules, review items.
- **Config:** Input `={{ $json }}`, structured output enabled.
- **Language model:** OpenAI GPT- (node name)
- **Input:** Financial Statement Generator Agent
- **Output:** Main → Tax Agent Communication Agent
- **Failure/edge cases:** tax rules are jurisdiction-specific; model output should be reviewed; schema mismatch risk.

#### Node: OpenAI GPT-
- **Type / role:** `lmChatOpenAi` for Tax Report Generator Agent.
- **Config:** Model `gpt-4o`, temperature `0.2`.

#### Node: Tax Report Schema
- **Type / role:** `outputParserStructured`
- **Schema:** taxableIncome (number), deductions[] {type, amount}, taxLiability {incomeTax, salesTax, payrollTax}, schedules[], itemsForReview[], taxPlanningOpportunities[].
- **Connected to:** Tax Report Generator Agent via `ai_outputParser`.

#### Node: Tax Agent Communication Agent
- **Type / role:** LangChain Agent — drafts an email and sends it using Gmail Tool.
- **Config:**
  - Input `={{ $json }}`
  - System message instructs to “Use the Gmail Tool to send the email…”
  - No explicit output parser
- **Language model:** OpenAI GPT-4
- **Tools:** Gmail Tool (connected via `ai_tool`)
- **Input:** Tax Report Generator Agent
- **Output:** Main → Aggregate All Results
- **Failure/edge cases:** Tool invocation may fail if Gmail credentials missing; agent must output tool fields expected by Gmail Tool (`emailSubject`, `emailBody`, `taxAgentEmail`) via `$fromAI()` mapping.

#### Node: OpenAI GPT-4
- **Type / role:** `lmChatOpenAi` for Tax Agent Communication Agent.
- **Config:** Model `gpt-4o`, temperature `0.2`.

#### Node: Gmail Tool
- **Type / role:** `gmailTool` — tool node callable by the agent to send emails.
- **Config (tool parameters):**
  - To: `={{ $fromAI('taxAgentEmail') }}`
  - Subject: `={{ $fromAI('emailSubject') }}`
  - Message: `={{ $fromAI('emailBody') }}`
- **Credentials:** Gmail OAuth2 “Gmail account”
- **Failure/edge cases:** OAuth token expiry, missing scopes (send), agent not providing required fields → tool execution error.

---

### Block 7 — Aggregation & Executive Summary Email
**Overview:** Aggregates outputs from “post journals” and “tax agent comms,” formats a simple final report payload, and emails the daily summary.

**Nodes involved:**
- Aggregate All Results
- Format Final Report
- Send Summary Email

#### Node: Aggregate All Results
- **Type / role:** `Aggregate` — consolidates multiple incoming branches into a single combined item.
- **Config:** `aggregateAllItemData`
- **Inputs:** Tax Agent Communication Agent, Post Journal Entries to System
- **Output:** Main → Format Final Report
- **Edge cases:** If upstream branches don’t both produce items (e.g., Check for Errors blocks the tax branch), aggregation may be incomplete or behave unexpectedly.

#### Node: Format Final Report
- **Type / role:** `Set` — creates final fields for summary email.
- **Config:**
  - reportTitle: “Autonomous Accounting System - Daily Report”
  - companyName: from Workflow Configuration
  - reportDate: `={{ $now.toFormat('yyyy-MM-dd') }}`
  - summary: “All accounting tasks completed successfully”
  - includeOtherFields: true
- **Output:** Main → Send Summary Email
- **Edge cases:** Summary is static; does not reflect errors detected unless you add conditional logic.

#### Node: Send Summary Email
- **Type / role:** `Gmail` — sends the final report to management/recipient.
- **Config:**
  - To: `={{ $('Workflow Configuration').first().json.summaryRecipientEmail }}`
  - Subject: `Autonomous Accounting System - Daily Report - {{ $now.toFormat('yyyy-MM-dd') }}`
  - HTML body includes title/company/date/status
- **Credentials:** Gmail OAuth2 “Gmail account”
- **Edge cases:** same Gmail auth/scope concerns; HTML rendering differences.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Accounting Run | scheduleTrigger | Daily workflow trigger | — | Workflow Configuration | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| Workflow Configuration | set | Centralized runtime configuration | Daily Accounting Run | Fetch Bank Transactions; Fetch Accounting System Data | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| Fetch Bank Transactions | httpRequest | Ingest bank transactions from API | Workflow Configuration | Combine Financial Data | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| Fetch Accounting System Data | httpRequest | Ingest accounting system records from API | Workflow Configuration | Combine Financial Data | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| Combine Financial Data | merge | Combine bank + accounting data | Fetch Bank Transactions; Fetch Accounting System Data | Transaction Classifier Agent | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| Transaction Classifier Agent | @n8n/n8n-nodes-langchain.agent | AI classification of transactions | Combine Financial Data | Account Reconciliation Agent | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| OpenAI GPT | lmChatOpenAi | LLM for Transaction Classifier Agent | — | Transaction Classifier Agent (ai_languageModel) | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| Transaction Classification Schema | outputParserStructured | Enforce structured classification output | — | Transaction Classifier Agent (ai_outputParser) | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| Account Reconciliation Agent | @n8n/n8n-nodes-langchain.agent | AI matching/reconciliation | Transaction Classifier Agent | Journal Entry Generator Agent | ## Reconciliation & Matching\n\n**What:** Builds reconciliation schemas and matches transactions to account records.\n**Why:** Automatically ensures accurate matching and surfaces discrepancies. |
| OpenAI | lmChatOpenAi | LLM for Account Reconciliation Agent | — | Account Reconciliation Agent (ai_languageModel) | ## Reconciliation & Matching\n\n**What:** Builds reconciliation schemas and matches transactions to account records.\n**Why:** Automatically ensures accurate matching and surfaces discrepancies. |
| Reconciliation Schema | outputParserStructured | Enforce structured reconciliation output | — | Account Reconciliation Agent (ai_outputParser) | ## Reconciliation & Matching\n\n**What:** Builds reconciliation schemas and matches transactions to account records.\n**Why:** Automatically ensures accurate matching and surfaces discrepancies. |
| Journal Entry Generator Agent | @n8n/n8n-nodes-langchain.agent | AI journal entry generation | Account Reconciliation Agent | Error Detection Agent; Post Journal Entries to System | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity. |
| OpenAI GPT-6 | lmChatOpenAi | LLM for Journal Entry Generator Agent | — | Journal Entry Generator Agent (ai_languageModel) | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity. |
| Journal Entry Schema | outputParserStructured | Enforce structured journal entry output | — | Journal Entry Generator Agent (ai_outputParser) | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity. |
| Error Detection Agent | @n8n/n8n-nodes-langchain.agent | AI validation/error detection for journals | Journal Entry Generator Agent | Check for Errors | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity. |
| Error Detection Schema | outputParserStructured | Enforce structured error report output | — | Error Detection Agent (ai_outputParser) | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity. |
| Check for Errors | if | Gate based on detected errors | Error Detection Agent | Financial Statement Generator Agent | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity. |
| Post Journal Entries to System | httpRequest | POST journals to external accounting API | Journal Entry Generator Agent | Aggregate All Results | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity. |
| Financial Statement Generator Agent | @n8n/n8n-nodes-langchain.agent | Generate financial statements/ratios | Check for Errors | Tax Report Generator Agent | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| OpenAI GPT-5 | lmChatOpenAi | LLM for Financial Statement Generator Agent | — | Financial Statement Generator Agent (ai_languageModel) | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Financial Statement Schema | outputParserStructured | Enforce structured financial statements | — | Financial Statement Generator Agent (ai_outputParser) | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Tax Report Generator Agent | @n8n/n8n-nodes-langchain.agent | Generate tax computations/schedules | Financial Statement Generator Agent | Tax Agent Communication Agent | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| OpenAI GPT- | lmChatOpenAi | LLM for Tax Report Generator Agent | — | Tax Report Generator Agent (ai_languageModel) | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Tax Report Schema | outputParserStructured | Enforce structured tax report | — | Tax Report Generator Agent (ai_outputParser) | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Tax Agent Communication Agent | @n8n/n8n-nodes-langchain.agent | Draft/send email to tax agent via tool | Tax Report Generator Agent | Aggregate All Results | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| OpenAI GPT-4 | lmChatOpenAi | LLM for Tax Agent Communication Agent | — | Tax Agent Communication Agent (ai_languageModel) | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Gmail Tool | gmailTool | Agent tool to send tax-agent email | — | Tax Agent Communication Agent (ai_tool) | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Aggregate All Results | aggregate | Consolidate branch outputs | Tax Agent Communication Agent; Post Journal Entries to System | Format Final Report | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Format Final Report | set | Prepare summary fields for email | Aggregate All Results | Send Summary Email | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Send Summary Email | gmail | Send executive daily report | Format Final Report | — | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |
| Calculator Tool | toolCalculator | Arithmetic tool for agents | — | Multiple agents (ai_tool) | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity. |
| Sticky Note | stickyNote | Canvas annotation | — | — | ## Prerequisites\nOpenAI API account with GPT-4 access\n## Use Cases\nMonthly financial close automation, daily transaction monitoring for fraud detection\n### Customization\nReplace Fable Bank with your banking API\n### Benefits\nReduces reconciliation time by 90%, eliminates manual data entry errors |
| Sticky Note1 | stickyNote | Canvas annotation | — | — | ## Setup Steps\n1. Configure Fable Bank API credentials for transaction data access \n2. Add OpenAI API key for GPT-4 classification and reconciliation models \n3. Set up NVIDIA NIM credentials for anomaly detection services \n4. Connect Google Sheets for reconciliation schema storage \n5. Configure Gmail account for automated report distribution |
| Sticky Note2 | stickyNote | Canvas annotation | — | — | ## How It Works\nThis workflow automates end-to-end financial transaction processing for finance teams managing high-volume bank data. It eliminates manual reconciliation by intelligently classifying transactions, detecting anomalies, and generating executive summaries. The system pulls transaction data from Fable Bank, routes it through multiple AI models (OpenAI GPT-4, NVIDIA NIM) for classification and analysis, reconciles accounts, and distributes formatted reports via email. Finance managers and accounting teams benefit from reduced processing time, improved accuracy, and real-time anomaly detection. The workflow handles transaction categorization, reconciliation schema generation, account matching, journal entry creation, and comprehensive reporting—transforming hours of manual work into minutes of automated processing with AI-enhanced accuracy. |
| Sticky Note3 | stickyNote | Canvas annotation | — | — | ## Journalization & Anomaly Detection\n\n**What:** Generates journal entries and detects anomalies with NVIDIA NIM analysis.\n**Why:** Preserves accounting integrity and flags suspicious activity.\n |
| Sticky Note4 | stickyNote | Canvas annotation | — | — | ## Reconciliation & Matching\n\n**What:** Builds reconciliation schemas and matches transactions to account records.\n**Why:** Automatically ensures accurate matching and surfaces discrepancies. |
| Sticky Note5 | stickyNote | Canvas annotation | — | — | ## Transaction Ingestion & Classification\n\n**What:** Pulls transaction data from the Fable Bank API and classifies it using AI agents.\n**Why:** Eliminates manual data collection and sorting through intelligent automation. |
| Sticky Note6 | stickyNote | Canvas annotation | — | — | ## Reporting & Distribution\n\n**What:** Compiles final reports and sends formatted summaries via Gmail.\n**Why:** Delivers actionable insights without manual report preparation. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named “AI-Powered Financial Transaction Reconciliation & Reporting System” (or your preferred title).

2. **Add Trigger**
   - Add **Schedule Trigger** node named **Daily Accounting Run**.
   - Set schedule to run daily at **02:00** (adjust timezone in n8n settings if needed).

3. **Add Configuration node**
   - Add **Set** node named **Workflow Configuration**.
   - Add string fields:
     - `bankApiUrl`, `accountingSystemApiUrl`, `journalEntryApiUrl`
     - `taxAgentEmail`, `summaryRecipientEmail`
     - `companyName`, `fiscalYearEnd`
   - Keep placeholders until you have real endpoints/emails.
   - Connect: **Daily Accounting Run → Workflow Configuration**.

4. **Add ingestion HTTP nodes**
   - Add **HTTP Request** node **Fetch Bank Transactions**
     - URL: `{{$('Workflow Configuration').first().json.bankApiUrl}}`
     - Add header `Content-Type: application/json`
     - Configure authentication (choose the credential type your bank API requires).
   - Add **HTTP Request** node **Fetch Accounting System Data**
     - URL: `{{$('Workflow Configuration').first().json.accountingSystemApiUrl}}`
     - Header `Content-Type: application/json`
     - Configure authentication for the accounting API.
   - Connect: **Workflow Configuration → Fetch Bank Transactions** and **Workflow Configuration → Fetch Accounting System Data**.

5. **Merge both datasets**
   - Add **Merge** node named **Combine Financial Data**
     - Mode: **Combine**
     - Combine By: **Combine All**
   - Connect:
     - **Fetch Bank Transactions → Combine Financial Data (Input 1)**
     - **Fetch Accounting System Data → Combine Financial Data (Input 2)**

6. **Create shared Agent tool**
   - Add **Calculator Tool** node (LangChain tool).

7. **Transaction classification agent**
   - Add **OpenAI Chat Model** node named **OpenAI GPT**
     - Model: `gpt-4o`, Temperature: `0.2`
     - Configure **OpenAI API credentials**.
   - Add **Structured Output Parser** node named **Transaction Classification Schema**
     - Paste the manual schema (object with transactionId/category/accountCode/etc.).
   - Add **AI Agent** node named **Transaction Classifier Agent**
     - Text: `{{$json}}`
     - System message: classifier instructions (as in workflow)
     - Enable structured output parser
     - Attach tools: **Calculator Tool**
     - Connect model: **OpenAI GPT → Transaction Classifier Agent** (ai_languageModel)
     - Connect parser: **Transaction Classification Schema → Transaction Classifier Agent** (ai_outputParser)
   - Connect: **Combine Financial Data → Transaction Classifier Agent**

8. **Reconciliation agent**
   - Add **OpenAI Chat Model** node named **OpenAI** (for reconciliation)
     - Model `gpt-4o`, temp `0.2`, same OpenAI credentials.
   - Add **Structured Output Parser** node named **Reconciliation Schema** (use the example-based schema).
   - Add **AI Agent** node **Account Reconciliation Agent**
     - Text `{{$json}}`, reconciliation system message, enable output parsing
     - Attach **Calculator Tool**
     - Connect: **OpenAI → Account Reconciliation Agent** (ai_languageModel)
     - Connect: **Reconciliation Schema → Account Reconciliation Agent** (ai_outputParser)
   - Connect: **Transaction Classifier Agent → Account Reconciliation Agent**

9. **Journal entry agent**
   - Add **OpenAI Chat Model** node named **OpenAI GPT-6**
   - Add **Structured Output Parser** node **Journal Entry Schema** (manual schema).
   - Add **AI Agent** node **Journal Entry Generator Agent**
     - Text `{{$json}}`, journalization system message, enable output parsing
     - Attach **Calculator Tool**
     - Connect LLM + parser accordingly
   - Connect: **Account Reconciliation Agent → Journal Entry Generator Agent**

10. **Error detection agent**
   - Add **OpenAI Chat Model** node (you may name it “OpenAI GPT-Error”) and connect it as `ai_languageModel` to **Error Detection Agent** (this is required for reliable execution).
   - Add **Structured Output Parser** node **Error Detection Schema** (manual schema).
   - Add **AI Agent** node **Error Detection Agent**
     - Text `{{$json}}`, error detection system message
     - Enable output parsing + attach Calculator Tool
   - Connect: **Journal Entry Generator Agent → Error Detection Agent**

11. **IF gate**
   - Add **IF** node **Check for Errors**
     - Condition 1: errors not empty: `{{$('Error Detection Agent').item.json.errors}}` notEmpty
     - Condition 2: array length > 0: same field lengthGt 0
   - Connect: **Error Detection Agent → Check for Errors**
   - Important: connect **both** true and false paths if you want reports regardless of errors.

12. **Post journal entries**
   - Add **HTTP Request** node **Post Journal Entries to System**
     - URL: `{{$('Workflow Configuration').first().json.journalEntryApiUrl}}`
     - Method: POST
     - Body: JSON `{{$json}}`
     - Header: `Content-Type: application/json`
     - Configure authentication credential for posting endpoint
   - Connect: **Journal Entry Generator Agent → Post Journal Entries to System**

13. **Financial statements**
   - Add **OpenAI Chat Model** node **OpenAI GPT-5**
   - Add **Structured Output Parser** node **Financial Statement Schema**
   - Add **AI Agent** node **Financial Statement Generator Agent**
     - Text `{{$json}}`, statements system message, enable parsing, attach Calculator Tool
   - Connect: **Check for Errors (true) → Financial Statement Generator Agent** (and optionally false too).

14. **Tax report**
   - Add **OpenAI Chat Model** node **OpenAI GPT-**
   - Add **Structured Output Parser** node **Tax Report Schema**
   - Add **AI Agent** node **Tax Report Generator Agent**
     - Text `{{$json}}`, tax system message, enable parsing, attach Calculator Tool
   - Connect: **Financial Statement Generator Agent → Tax Report Generator Agent**

15. **Tax agent email via tool**
   - Add **Gmail Tool** node
     - Configure Gmail OAuth2 credentials with send permissions
     - Set:
       - To: `{{$fromAI('taxAgentEmail')}}`
       - Subject: `{{$fromAI('emailSubject')}}`
       - Message: `{{$fromAI('emailBody')}}`
   - Add **OpenAI Chat Model** node **OpenAI GPT-4**
   - Add **AI Agent** node **Tax Agent Communication Agent**
     - Text `{{$json}}`
     - System message instructing to email the tax agent via Gmail tool
     - Attach tool: **Gmail Tool**
     - Connect model: **OpenAI GPT-4 → Tax Agent Communication Agent** (ai_languageModel)
   - Connect: **Tax Report Generator Agent → Tax Agent Communication Agent**

16. **Aggregate results**
   - Add **Aggregate** node **Aggregate All Results**
     - Mode: aggregate all item data
   - Connect: **Tax Agent Communication Agent → Aggregate All Results**
   - Connect: **Post Journal Entries to System → Aggregate All Results**

17. **Format final report**
   - Add **Set** node **Format Final Report**
     - reportTitle: fixed string
     - companyName: `{{$('Workflow Configuration').first().json.companyName}}`
     - reportDate: `{{$now.toFormat('yyyy-MM-dd')}}`
     - summary: fixed string (or make conditional)
   - Connect: **Aggregate All Results → Format Final Report**

18. **Send executive summary**
   - Add **Gmail** node **Send Summary Email**
     - To: `{{$('Workflow Configuration').first().json.summaryRecipientEmail}}`
     - Subject: `Autonomous Accounting System - Daily Report - {{$now.toFormat('yyyy-MM-dd')}}`
     - Message: HTML template using reportTitle/companyName/reportDate/summary
     - Configure Gmail OAuth2 credentials
   - Connect: **Format Final Report → Send Summary Email**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI API account with GPT-4 access | Prerequisites |
| Use cases: monthly close automation; daily transaction monitoring for fraud detection | Prerequisites / Use Cases |
| Customization: “Replace Fable Bank with your banking API” | Customization note |
| Benefits: “Reduces reconciliation time by 90%, eliminates manual data entry errors” | Value statement |
| Setup Steps: configure bank API creds; OpenAI key; NVIDIA NIM creds; Google Sheets for schema storage; Gmail for distribution | Sticky note content (note: NVIDIA NIM + Google Sheets are mentioned but **not implemented** as nodes in this workflow) |
| “How It Works” narrative describing end-to-end automation and mentions NVIDIA NIM | High-level description (NVIDIA NIM not present in nodes) |

