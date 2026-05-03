Reconcile Stripe, bank, and e-commerce data with GPT-4.1 and Google Sheets

https://n8nworkflows.xyz/workflows/reconcile-stripe--bank--and-e-commerce-data-with-gpt-4-1-and-google-sheets-12493


# Reconcile Stripe, bank, and e-commerce data with GPT-4.1 and Google Sheets

## 1. Workflow Overview

**Workflow name (in JSON):** Intelligent financial reconciliation and tax reporting automation  
**Provided title:** Reconcile Stripe, bank, and e-commerce data with GPT-4.1 and Google Sheets

**Purpose:**  
Automatically pull financial data from Stripe, a bank feed API, an invoice system API, and an e-commerce platform API; combine it; use an AI “orchestrator” (GPT-4.1-mini via n8n’s LangChain nodes) to coordinate specialized agents that (1) detect mismatches, (2) analyze root causes, and (3) generate ledger correction instructions; then log results to Google Sheets and send an email notification.

**Primary use cases (from sticky notes):**
- Monthly financial close automation
- Daily transaction reconciliation
- High-volume transaction businesses wanting faster, more consistent reconciliation

### 1.1 Scheduled Trigger & Configuration
Runs on a schedule and sets all environment-like variables (API endpoints, thresholds, notification email).

### 1.2 Data Collection (Stripe + External APIs)
Fetches transactions from Stripe and pulls JSON from bank, invoice, and e-commerce endpoints.

### 1.3 Data Aggregation
Combines outputs from all sources into one unified payload.

### 1.4 AI Orchestration (Multi-agent reconciliation)
An orchestrator agent calls three agent tools (Mismatch Detection, Root Cause Analysis, Ledger Correction) and uses a calculator tool as needed, enforcing structured outputs via JSON schema parsers.

### 1.5 Apply/Format Corrections + Logging & Notifications
Transforms orchestrator output into ledger-entry-like records, appends/updates Google Sheets, and emails a summary.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Trigger & Workflow Configuration

**Overview:**  
Starts the workflow on a daily schedule and defines key variables (API URLs, reconciliation threshold, notification email) used by later nodes.

**Nodes involved:**
- Schedule Trigger
- Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration (interpreted):** Runs at **02:00** (server/workspace timezone) based on an “interval” rule with `triggerAtHour: 2`.
- **Connections:**
  - **Output →** Workflow Configuration
- **Potential failures / edge cases:**
  - Timezone mismatch vs accounting period cutoffs (e.g., bank day close).
  - If n8n instance is down at 02:00, execution depends on n8n schedule behavior (missed run vs catch-up).
- **Version notes:** typeVersion `1.3`.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — defines reusable configuration values.
- **Configuration (interpreted):**
  - Sets:
    - `stripeApiUrl` (placeholder; not actually used elsewhere in this JSON)
    - `bankFeedApiUrl` (used by Get Bank Feed Data)
    - `invoiceApiUrl` (used by Get Invoice Data)
    - `ecommerceApiUrl` (used by Get Ecommerce Platform Data)
    - `taxAgentApiUrl` (placeholder; referenced in email, but no submission node exists)
    - `reconciliationThreshold` = `0.01`
    - `notificationEmail` (used by Gmail node)
  - “Include other fields” enabled, so incoming fields are preserved (though trigger provides minimal data).
- **Key expressions/variables:** Values are literal placeholders except the number threshold.
- **Connections:**
  - **Input:** Schedule Trigger
  - **Output → (fan-out):** Get Stripe Transactions, Get Bank Feed Data, Get Invoice Data, Get Ecommerce Platform Data
- **Potential failures / edge cases:**
  - Placeholders not replaced will cause downstream HTTP request failures.
  - Threshold is defined but not explicitly injected into the AI tools; unless the AI infers it from the combined JSON, mismatches may not respect it consistently.
- **Version notes:** typeVersion `3.4`.

**Sticky note coverage (context):**
- **“Scheduled Data Collection”**: emphasizes automated retrieval and consistent timing.
- **“How It Works”**: high-level description of the entire workflow.
- **“Setup Steps”**: credential/config reminders.

---

### Block 2 — Data Collection (Stripe + Bank/Invoice/E-commerce)

**Overview:**  
Pulls raw transaction and sales/financial records from four sources. Stripe uses the native Stripe node; the other three use HTTP requests to external endpoints.

**Nodes involved:**
- Get Stripe Transactions
- Get Bank Feed Data
- Get Invoice Data
- Get Ecommerce Platform Data

#### Node: Get Stripe Transactions
- **Type / role:** `n8n-nodes-base.stripe` — retrieves Stripe objects.
- **Configuration (interpreted):**
  - **Resource:** `charge`
  - **Operation:** `getAll`
  - **Limit:** 100
- **Connections:**
  - **Input:** Workflow Configuration
  - **Output →** Combine All Financial Data
- **Potential failures / edge cases:**
  - Stripe credential missing/invalid → auth errors.
  - Limit 100 may be insufficient for daily/monthly close; pagination beyond 100 not configured here.
  - “Charges” may not align with payouts/bank deposits; reconciliation often needs balance transactions/payouts.
- **Version notes:** typeVersion `1`.

#### Node: Get Bank Feed Data
- **Type / role:** `n8n-nodes-base.httpRequest` — calls bank feed API endpoint.
- **Configuration (interpreted):**
  - **URL:** `={{ $('Workflow Configuration').first().json.bankFeedApiUrl }}`
  - Sends header `Content-Type: application/json`
  - Authentication not configured in-node (no auth parameters shown); assumes endpoint is open, uses other defaults, or is configured elsewhere.
- **Connections:**
  - **Input:** Workflow Configuration
  - **Output →** Combine All Financial Data
- **Potential failures / edge cases:**
  - Missing/invalid URL placeholder.
  - Bank APIs usually require OAuth/API keys; without auth this will likely 401/403.
  - Response shape differences (array vs object) can complicate downstream AI comparison.
- **Version notes:** typeVersion `4.3`.

#### Node: Get Invoice Data
- **Type / role:** `n8n-nodes-base.httpRequest` — calls invoicing system API endpoint.
- **Configuration (interpreted):**
  - **URL:** `={{ $('Workflow Configuration').first().json.invoiceApiUrl }}`
  - Header: `Content-Type: application/json`
- **Connections:**
  - **Input:** Workflow Configuration
  - **Output →** Combine All Financial Data
- **Potential failures / edge cases:** same class as above (auth, URL placeholders, unpredictable schema).
- **Version notes:** typeVersion `4.3`.

#### Node: Get Ecommerce Platform Data
- **Type / role:** `n8n-nodes-base.httpRequest` — calls e-commerce platform API endpoint.
- **Configuration (interpreted):**
  - **URL:** `={{ $('Workflow Configuration').first().json.ecommerceApiUrl }}`
  - Header: `Content-Type: application/json`
- **Connections:**
  - **Input:** Workflow Configuration
  - **Output →** Combine All Financial Data
- **Potential failures / edge cases:** same class as above, plus:
  - Shopify/WooCommerce APIs often paginate and require date filters; neither is configured.
- **Version notes:** typeVersion `4.3`.

**Sticky note coverage (context):**
- **“Scheduled Data Collection”**: applies to these retrieval nodes and their trigger/config.

---

### Block 3 — Combine All Financial Data

**Overview:**  
Aggregates the four upstream node outputs into a single combined dataset for the AI orchestrator.

**Nodes involved:**
- Combine All Financial Data

#### Node: Combine All Financial Data
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates multiple inputs.
- **Configuration (interpreted):**
  - Mode: “aggregate all item data” (`aggregateAllItemData`), which merges/collects item data from all incoming streams into a single item/payload.
- **Connections:**
  - **Inputs:** Get Stripe Transactions, Get Bank Feed Data, Get Invoice Data, Get Ecommerce Platform Data
  - **Output →** Orchestrator Agent
- **Potential failures / edge cases:**
  - If any upstream returns multiple items vs single item, the aggregate result can become large and inconsistent.
  - Very large payloads can exceed LLM context limits or slow execution.
- **Version notes:** typeVersion `1`.

---

### Block 4 — AI Orchestration (Agents + Tools + Structured Parsers)

**Overview:**  
Uses a main “Orchestrator Agent” to coordinate three specialized agent tools. Each tool is backed by its own OpenAI chat model node and returns structured JSON enforced by output parsers. A calculator tool is available to the orchestrator.

**Nodes involved:**
- Orchestrator Agent
- Mismatch Detection Agent Tool
- Root Cause Analysis Agent Tool
- Ledger Correction Agent Tool
- Calculator Tool
- OpenAI Model - Orchestrator
- OpenAI Model - Mismatch Detection
- OpenAI Model - Root Cause
- OpenAI Model - Ledger Correction
- Orchestrator Output Parser
- Mismatch Detection Output Parser
- Root Cause Output Parser
- Ledger Correction Output Parser

#### Node: Orchestrator Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — central agent that can call tools.
- **Configuration (interpreted):**
  - **Input text:** `={{ $json }}` (entire aggregated JSON fed as the agent prompt content)
  - **System message:** instructs it to:
    1. Call Mismatch Detection Agent Tool
    2. For each mismatch, call Root Cause Analysis Agent Tool
    3. Call Ledger Correction Agent Tool
    4. Return structured JSON with mismatches, root causes, corrections
    - Also: “Use the Calculator Tool for calculations.”
  - **Structured output enabled:** `hasOutputParser: true` and connected to Orchestrator Output Parser.
- **Connections:**
  - **Main input:** Combine All Financial Data
  - **AI language model input:** OpenAI Model - Orchestrator
  - **AI tools available:** Mismatch Detection Agent Tool, Root Cause Analysis Agent Tool, Ledger Correction Agent Tool, Calculator Tool
  - **AI output parser:** Orchestrator Output Parser
  - **Main output →** Apply Ledger Corrections
- **Potential failures / edge cases:**
  - Tool-call chain may be incomplete: orchestrator may skip tools if prompt/context is unclear.
  - Large input JSON may cause truncation/poor reasoning.
  - Output parser can fail if the model returns non-conforming JSON.
- **Version notes:** typeVersion `3.1`.

#### Node: Mismatch Detection Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool callable by orchestrator.
- **Configuration (interpreted):**
  - **Input text:** `={{ $fromAI('financialData', 'All financial data from different sources', 'json') }}`
    - Expects the orchestrator to pass `financialData` as JSON.
  - **System message:** compare across sources; flag mismatches; use threshold (but threshold value is not explicitly passed unless included in `financialData`).
  - **Structured output:** enabled + connected to Mismatch Detection Output Parser.
- **Connections:**
  - **AI language model:** OpenAI Model - Mismatch Detection
  - **AI output parser:** Mismatch Detection Output Parser
  - **Tool available to:** Orchestrator Agent (as `ai_tool`)
- **Potential failures / edge cases:**
  - If orchestrator doesn’t pass `financialData` key, `$fromAI(...)` can be empty/invalid.
  - Schema mismatch in tool output causes parser failure.
- **Version notes:** typeVersion `3`.

#### Node: Root Cause Analysis Agent Tool
- **Type / role:** `agentTool` — analyzes mismatches.
- **Configuration:**
  - Input: `={{ $fromAI('mismatchData', 'Mismatch information to analyze', 'json') }}`
  - Produces root cause with confidence + category.
  - Structured output via Root Cause Output Parser.
- **Connections:**
  - AI model: OpenAI Model - Root Cause
  - Output parser: Root Cause Output Parser
  - Tool available to Orchestrator Agent
- **Edge cases:**
  - Missing `mismatchData` from orchestrator.
  - Confidence/category constrained by enum in parser; model must comply.
- **Version notes:** typeVersion `3`.

#### Node: Ledger Correction Agent Tool
- **Type / role:** `agentTool` — generates double-entry correction instructions.
- **Configuration:**
  - Input: `={{ $fromAI('rootCauseData', 'Root cause analysis results', 'json') }}`
  - Structured output via Ledger Correction Output Parser.
- **Connections:**
  - AI model: OpenAI Model - Ledger Correction
  - Output parser: Ledger Correction Output Parser
  - Tool available to Orchestrator Agent
- **Edge cases:**
  - Missing `rootCauseData`.
  - Output schema requires entries with `account`, `debit`, `credit`, `description`; model may omit or misformat.
- **Version notes:** typeVersion `3`.

#### Node: Calculator Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCalculator` — arithmetic helper tool.
- **Connections:**
  - Tool available to Orchestrator Agent
- **Edge cases:** limited to arithmetic; doesn’t validate accounting logic.
- **Version notes:** typeVersion `1`.

#### Nodes: OpenAI Model - Orchestrator / Mismatch Detection / Root Cause / Ledger Correction
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider nodes.
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini` (selected from list)
  - Credentials: “OpenAi account”
- **Connections:**
  - Each model node connects as `ai_languageModel` to its respective agent/tool.
- **Potential failures / edge cases:**
  - Invalid OpenAI API key, quota exceeded, rate limiting.
  - Model name availability depends on account/region.
- **Version notes:** typeVersion `1.3`.

#### Nodes: Output Parsers (Structured)
All are `@n8n/n8n-nodes-langchain.outputParserStructured` with **manual JSON schemas**:

- **Mismatch Detection Output Parser**: expects `{ mismatches: [ { mismatchId, transactionId, sources, expectedAmount, actualAmount, discrepancyAmount, transactionDate, description } ] }`
- **Root Cause Output Parser**: expects `{ rootCauseAnalysis: [ { mismatchId, rootCause, explanation, confidence (high/medium/low), category (timing/fees/refund/error/duplicate/missing/other) } ] }`
- **Ledger Correction Output Parser**: expects `{ corrections: [ { mismatchId, correctionType, entries:[{account,debit,credit,description}], reference, notes } ] }`
- **Orchestrator Output Parser**: expects:
  - `reconciliationSummary` (totalMismatches, totalCorrections, reconciliationDate)
  - `mismatches` array items containing `mismatchId, sources, discrepancyAmount, rootCause, confidence, correction`

**Connections:**
- Each parser is wired to its respective agent/tool via `ai_outputParser`.

**Edge cases:**
- Any deviation from schema → parser failure → node error.
- Orchestrator schema does **not** require `transactionId` in `mismatches`, but downstream code tries to read it (see Block 5).

**Sticky note coverage (context):**
- **“AI-Powered Mismatch Detection”**: applies to mismatch tool + model + parser.
- **“Root Cause Analysis”**: applies to root-cause tool + model + parser.

---

### Block 5 — Apply Corrections, Log to Sheets, Send Email

**Overview:**  
Transforms the orchestrator’s structured output into a ledger-entry list and summary, logs results to Google Sheets, and emails a completion message.

**Nodes involved:**
- Apply Ledger Corrections
- Log Fixes to Google Sheets
- Send Notification Email

#### Node: Apply Ledger Corrections
- **Type / role:** `n8n-nodes-base.code` — post-processing and normalization.
- **Configuration (interpreted):**
  - Reads first input item as `reconciliationData`.
  - Treats `reconciliationData.mismatches` as “corrections”.
  - Maps each mismatch into a `ledgerEntries` object with:
    - mismatchId
    - transactionId: `mismatch.transactionId || mismatch.mismatchId` (note: orchestrator output schema doesn’t include transactionId)
    - rootCause, confidence, correction, discrepancyAmount
    - processedAt timestamp
    - status = `applied` (no external ledger write actually happens; it’s a status label)
  - Creates `reconciliationSummary` plus `totalDiscrepancyAmount`.
  - Outputs: `{ reconciliationSummary, ledgerEntries, originalData }`
- **Connections:**
  - **Input:** Orchestrator Agent
  - **Outputs →** Log Fixes to Google Sheets AND Send Notification Email
- **Potential failures / edge cases:**
  - If orchestrator returns a different structure, `reconciliationData.mismatches` may be undefined → defaults to empty list (silent “success” but no entries).
  - “Applied” is not real posting to an accounting system; it only formats data.
- **Version notes:** typeVersion `2`.

#### Node: Log Fixes to Google Sheets
- **Type / role:** `n8n-nodes-base.googleSheets` — append/update reconciliation output.
- **Configuration (interpreted):**
  - Operation: `appendOrUpdate`
  - Document: placeholder Google Sheets document ID
  - Sheet name: `Reconciliation Log`
  - Mapping mode: auto-map input data to columns
- **Connections:**
  - **Input:** Apply Ledger Corrections
- **Potential failures / edge cases:**
  - Sheet columns must match the flattened structure; nested objects like `ledgerEntries` may not map cleanly without prior flattening.
  - AppendOrUpdate typically needs a matching key column; not specified here—behavior may degrade to append-only or fail depending on node defaults.
  - OAuth scope/permission issues.
- **Version notes:** typeVersion `4.7`.

#### Node: Send Notification Email
- **Type / role:** `n8n-nodes-base.gmail` — sends email summary.
- **Configuration (interpreted):**
  - To: `={{ $('Workflow Configuration').first().json.notificationEmail }}`
  - Subject: `Financial Reconciliation Complete - <local date>`
  - Body includes:
    - total mismatches + corrections from Apply Ledger Corrections output
    - “Report Submitted” value from `Submit Report to Tax Agent` node (but that node does not exist in this workflow JSON)
- **Connections:**
  - **Input:** Apply Ledger Corrections
- **Potential failures / edge cases:**
  - **Expression references missing node:** `$('Submit Report to Tax Agent')...` will error at runtime because the node is not present. This is currently a blocking defect for the email node.
  - Gmail OAuth invalid/expired.
- **Version notes:** typeVersion `2.2`.

**Sticky note coverage (context):**
- **“Automated Ledger Corrections & Email”**: applies to Apply Ledger Corrections + Google Sheets + Gmail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled entry point (02:00 run) | — | Workflow Configuration | ## Scheduled Data Collection\n**Why:** Automated retrieval ensures consistent, timely reconciliation without manual intervention, capturing transactions from all financial sources simultaneously. |
| Workflow Configuration | set | Centralized config variables (URLs, threshold, email) | Schedule Trigger | Get Stripe Transactions; Get Bank Feed Data; Get Invoice Data; Get Ecommerce Platform Data | ## Scheduled Data Collection\n**Why:** Automated retrieval ensures consistent, timely reconciliation without manual intervention, capturing transactions from all financial sources simultaneously. |
| Get Stripe Transactions | stripe | Fetch Stripe charges (limit 100) | Workflow Configuration | Combine All Financial Data | ## Scheduled Data Collection\n**Why:** Automated retrieval ensures consistent, timely reconciliation without manual intervention, capturing transactions from all financial sources simultaneously. |
| Get Bank Feed Data | httpRequest | Pull bank feed JSON from configured endpoint | Workflow Configuration | Combine All Financial Data | ## Scheduled Data Collection\n**Why:** Automated retrieval ensures consistent, timely reconciliation without manual intervention, capturing transactions from all financial sources simultaneously. |
| Get Invoice Data | httpRequest | Pull invoice system JSON from configured endpoint | Workflow Configuration | Combine All Financial Data | ## Scheduled Data Collection\n**Why:** Automated retrieval ensures consistent, timely reconciliation without manual intervention, capturing transactions from all financial sources simultaneously. |
| Get Ecommerce Platform Data | httpRequest | Pull e-commerce platform JSON from configured endpoint | Workflow Configuration | Combine All Financial Data | ## Scheduled Data Collection\n**Why:** Automated retrieval ensures consistent, timely reconciliation without manual intervention, capturing transactions from all financial sources simultaneously. |
| Combine All Financial Data | aggregate | Aggregate all upstream data into one payload | Get Stripe Transactions; Get Bank Feed Data; Get Invoice Data; Get Ecommerce Platform Data | Orchestrator Agent | ## How It Works\nThis workflow automates financial reconciliation by orchestrating multiple AI agents to detect mismatches, analyze root causes, and apply corrections across bank statements, invoices, and e-commerce platforms. Designed for finance teams, accountants, and business owners managing high transaction volumes, it eliminates manual reconciliation tedious work that typically consumes hours weekly. The system retrieves financial data from Stripe, banking APIs, and e-commerce platforms, then feeds it to specialized AI agents: one detects discrepancies using pattern recognition, another performs root cause analysis, and a third generates ledger corrections. An orchestrator agent coordinates these specialists, ensuring systematic processing. Results are logged to Google Sheets and trigger email notifications for critical issues, creating an audit trail while reducing reconciliation time from hours to minutes with 95%+ accuracy. |
| Orchestrator Agent | langchain.agent | Coordinates tools to detect mismatches, find root causes, propose corrections | Combine All Financial Data | Apply Ledger Corrections | ## Prerequisites\nNVIDIA API access, OpenAI API key, Stripe account\n## Use Cases\nMonthly financial close automation, daily transaction reconciliation\n## Customization\nModify detection thresholds, add custom financial data sources\n## Benefits\nReduces reconciliation time by 90%, eliminates manual data entry errors |
| Mismatch Detection Agent Tool | langchain.agentTool | Tool: compare sources and output mismatch list | (Called by Orchestrator Agent) | (Returns to Orchestrator Agent) | ## AI-Powered Mismatch Detection\n**Why:** Machine learning identifies discrepancies faster than manual review, catching subtle inconsistencies humans might miss across thousands of transactions. |
| Root Cause Analysis Agent Tool | langchain.agentTool | Tool: explain causes + confidence/category | (Called by Orchestrator Agent) | (Returns to Orchestrator Agent) | ## Root Cause Analysis\n**Why:** Understanding why mismatches occur prevents recurring errors and informs process improvements beyond simple correction. |
| Ledger Correction Agent Tool | langchain.agentTool | Tool: produce double-entry correction instructions | (Called by Orchestrator Agent) | (Returns to Orchestrator Agent) | ## Automated Ledger Corrections & Email\n**Why:** Direct correction generation eliminates transcription errors and accelerates resolution, maintaining accounting accuracy. |
| Calculator Tool | langchain.toolCalculator | Tool: arithmetic support for agent(s) | (Called by Orchestrator Agent) | (Returns to Orchestrator Agent) | ## Prerequisites\nNVIDIA API access, OpenAI API key, Stripe account\n## Use Cases\nMonthly financial close automation, daily transaction reconciliation\n## Customization\nModify detection thresholds, add custom financial data sources\n## Benefits\nReduces reconciliation time by 90%, eliminates manual data entry errors |
| OpenAI Model - Orchestrator | lmChatOpenAi | LLM backend for orchestrator agent | — | Orchestrator Agent (ai_languageModel) | ## Setup Steps\n1. Configure Stripe API credentials in \"Get Stripe Transactions\" node\n2. Add banking API authentication for \"Get Bank Feed Data\" node\n3. Connect e-commerce platform (Shopify/WooCommerce) credentials  \n4. Input NVIDIA API key for all OpenAI Model nodes\n5. Set OpenAI API key in Orchestrator Agent\n6. Configure Gmail credentials for notification node |
| OpenAI Model - Mismatch Detection | lmChatOpenAi | LLM backend for mismatch tool | — | Mismatch Detection Agent Tool (ai_languageModel) | ## AI-Powered Mismatch Detection\n**Why:** Machine learning identifies discrepancies faster than manual review, catching subtle inconsistencies humans might miss across thousands of transactions. |
| OpenAI Model - Root Cause | lmChatOpenAi | LLM backend for root cause tool | — | Root Cause Analysis Agent Tool (ai_languageModel) | ## Root Cause Analysis\n**Why:** Understanding why mismatches occur prevents recurring errors and informs process improvements beyond simple correction. |
| OpenAI Model - Ledger Correction | lmChatOpenAi | LLM backend for ledger correction tool | — | Ledger Correction Agent Tool (ai_languageModel) | ## Automated Ledger Corrections & Email\n**Why:** Direct correction generation eliminates transcription errors and accelerates resolution, maintaining accounting accuracy. |
| Orchestrator Output Parser | outputParserStructured | Enforces orchestrator final JSON schema | — | Orchestrator Agent (ai_outputParser) | ## Prerequisites\nNVIDIA API access, OpenAI API key, Stripe account\n## Use Cases\nMonthly financial close automation, daily transaction reconciliation\n## Customization\nModify detection thresholds, add custom financial data sources\n## Benefits\nReduces reconciliation time by 90%, eliminates manual data entry errors |
| Mismatch Detection Output Parser | outputParserStructured | Enforces mismatch list schema | — | Mismatch Detection Agent Tool (ai_outputParser) | ## AI-Powered Mismatch Detection\n**Why:** Machine learning identifies discrepancies faster than manual review, catching subtle inconsistencies humans might miss across thousands of transactions. |
| Root Cause Output Parser | outputParserStructured | Enforces root cause schema (enum confidence/category) | — | Root Cause Analysis Agent Tool (ai_outputParser) | ## Root Cause Analysis\n**Why:** Understanding why mismatches occur prevents recurring errors and informs process improvements beyond simple correction. |
| Ledger Correction Output Parser | outputParserStructured | Enforces ledger corrections schema | — | Ledger Correction Agent Tool (ai_outputParser) | ## Automated Ledger Corrections & Email\n**Why:** Direct correction generation eliminates transcription errors and accelerates resolution, maintaining accounting accuracy. |
| Apply Ledger Corrections | code | Converts mismatches into ledgerEntries + summary | Orchestrator Agent | Log Fixes to Google Sheets; Send Notification Email | ## Automated Ledger Corrections & Email\n**Why:** Direct correction generation eliminates transcription errors and accelerates resolution, maintaining accounting accuracy. |
| Log Fixes to Google Sheets | googleSheets | Append/update reconciliation results to sheet | Apply Ledger Corrections | — | ## Automated Ledger Corrections & Email\n**Why:** Direct correction generation eliminates transcription errors and accelerates resolution, maintaining accounting accuracy. |
| Send Notification Email | gmail | Email summary of reconciliation run | Apply Ledger Corrections | — | ## Automated Ledger Corrections & Email\n**Why:** Direct correction generation eliminates transcription errors and accelerates resolution, maintaining accounting accuracy. |
| Sticky Note | stickyNote | Comment: prerequisites/use cases/benefits | — | — | ## Prerequisites\nNVIDIA API access, OpenAI API key, Stripe account\n## Use Cases\nMonthly financial close automation, daily transaction reconciliation\n## Customization\nModify detection thresholds, add custom financial data sources\n## Benefits\nReduces reconciliation time by 90%, eliminates manual data entry errors |
| Sticky Note1 | stickyNote | Comment: setup steps | — | — | ## Setup Steps\n1. Configure Stripe API credentials in \"Get Stripe Transactions\" node\n2. Add banking API authentication for \"Get Bank Feed Data\" node\n3. Connect e-commerce platform (Shopify/WooCommerce) credentials  \n4. Input NVIDIA API key for all OpenAI Model nodes\n5. Set OpenAI API key in Orchestrator Agent\n6. Configure Gmail credentials for notification node |
| Sticky Note2 | stickyNote | Comment: end-to-end explanation | — | — | ## How It Works\nThis workflow automates financial reconciliation by orchestrating multiple AI agents to detect mismatches, analyze root causes, and apply corrections across bank statements, invoices, and e-commerce platforms. Designed for finance teams, accountants, and business owners managing high transaction volumes, it eliminates manual reconciliation tedious work that typically consumes hours weekly. The system retrieves financial data from Stripe, banking APIs, and e-commerce platforms, then feeds it to specialized AI agents: one detects discrepancies using pattern recognition, another performs root cause analysis, and a third generates ledger corrections. An orchestrator agent coordinates these specialists, ensuring systematic processing. Results are logged to Google Sheets and trigger email notifications for critical issues, creating an audit trail while reducing reconciliation time from hours to minutes with 95%+ accuracy. |
| Sticky Note3 | stickyNote | Comment: root cause rationale | — | — | ## Root Cause Analysis\n**Why:** Understanding why mismatches occur prevents recurring errors and informs process improvements beyond simple correction. |
| Sticky Note4 | stickyNote | Comment: mismatch detection rationale | — | — | ## AI-Powered Mismatch Detection\n**Why:** Machine learning identifies discrepancies faster than manual review, catching subtle inconsistencies humans might miss across thousands of transactions. |
| Sticky Note5 | stickyNote | Comment: scheduled collection rationale | — | — | ## Scheduled Data Collection\n**Why:** Automated retrieval ensures consistent, timely reconciliation without manual intervention, capturing transactions from all financial sources simultaneously. |
| Sticky Note6 | stickyNote | Comment: corrections/email rationale | — | — | ## Automated Ledger Corrections & Email\n**Why:** Direct correction generation eliminates transcription errors and accelerates resolution, maintaining accounting accuracy. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule Trigger” (Schedule Trigger node)**
   - Set it to run daily at **02:00** (or your desired hour).

2) **Create “Workflow Configuration” (Set node)**
   - Add fields:
     - `bankFeedApiUrl` (string)
     - `invoiceApiUrl` (string)
     - `ecommerceApiUrl` (string)
     - `taxAgentApiUrl` (string) *(optional unless you add a tax submission node)*
     - `reconciliationThreshold` (number, e.g. `0.01`)
     - `notificationEmail` (string)
   - Enable “Include other fields”.

3) **Connect Schedule Trigger → Workflow Configuration**

4) **Create “Get Stripe Transactions” (Stripe node)**
   - Resource: **Charge**
   - Operation: **Get All**
   - Limit: **100** (increase/add pagination if needed)
   - Configure **Stripe credentials** in n8n.
5) **Connect Workflow Configuration → Get Stripe Transactions**

6) **Create “Get Bank Feed Data” (HTTP Request node)**
   - URL expression: `{{ $('Workflow Configuration').first().json.bankFeedApiUrl }}`
   - Add header: `Content-Type: application/json`
   - Add authentication (API key/OAuth2) as required by your bank feed provider.
7) **Connect Workflow Configuration → Get Bank Feed Data**

8) **Create “Get Invoice Data” (HTTP Request node)**
   - URL expression: `{{ $('Workflow Configuration').first().json.invoiceApiUrl }}`
   - Header: `Content-Type: application/json`
   - Add required authentication.
9) **Connect Workflow Configuration → Get Invoice Data**

10) **Create “Get Ecommerce Platform Data” (HTTP Request node)**
   - URL expression: `{{ $('Workflow Configuration').first().json.ecommerceApiUrl }}`
   - Header: `Content-Type: application/json`
   - Add required authentication (Shopify/WooCommerce/etc.).
11) **Connect Workflow Configuration → Get Ecommerce Platform Data**

12) **Create “Combine All Financial Data” (Aggregate node)**
   - Operation/mode: **Aggregate All Item Data**
13) **Connect each data node → Combine All Financial Data**
   - Get Stripe Transactions → Combine
   - Get Bank Feed Data → Combine
   - Get Invoice Data → Combine
   - Get Ecommerce Platform Data → Combine

14) **Create the AI model nodes (4x “OpenAI Chat Model” via LangChain)**
   - Names:
     - OpenAI Model - Orchestrator
     - OpenAI Model - Mismatch Detection
     - OpenAI Model - Root Cause
     - OpenAI Model - Ledger Correction
   - Model: **gpt-4.1-mini**
   - Configure **OpenAI API credentials** in n8n.

15) **Create structured output parsers (4x Output Parser Structured)**
   - Names:
     - Orchestrator Output Parser
     - Mismatch Detection Output Parser
     - Root Cause Output Parser
     - Ledger Correction Output Parser
   - Schema type: **Manual**
   - Paste the corresponding JSON schemas (as per the workflow’s intent).

16) **Create tools (3x Agent Tool + 1x Calculator Tool)**
   - Create:
     - Mismatch Detection Agent Tool (with system message for mismatch detection)
     - Root Cause Analysis Agent Tool (with forensics system message)
     - Ledger Correction Agent Tool (with bookkeeping system message)
     - Calculator Tool
   - For each Agent Tool:
     - Connect its **AI language model** input to the corresponding OpenAI model node.
     - Connect its **AI output parser** to its corresponding parser node.
     - Ensure the tool’s input uses `$fromAI(...)` with the same key names you will pass from the orchestrator:
       - `financialData`, `mismatchData`, `rootCauseData`

17) **Create “Orchestrator Agent” (LangChain Agent node)**
   - Text input: `{{ $json }}` (from aggregated data)
   - System message: instructs it to call the three tools in sequence and return final structured JSON.
   - Connect:
     - **AI language model:** OpenAI Model - Orchestrator
     - **AI output parser:** Orchestrator Output Parser
     - **AI tools:** attach Mismatch Detection Agent Tool, Root Cause Analysis Agent Tool, Ledger Correction Agent Tool, Calculator Tool
18) **Connect Combine All Financial Data → Orchestrator Agent**

19) **Create “Apply Ledger Corrections” (Code node)**
   - Paste logic to:
     - read orchestrator output
     - map mismatches into `ledgerEntries`
     - compute reconciliation summary totals
20) **Connect Orchestrator Agent → Apply Ledger Corrections**

21) **Create “Log Fixes to Google Sheets” (Google Sheets node)**
   - Operation: **Append or Update**
   - Spreadsheet: select your document
   - Sheet: **Reconciliation Log**
   - Auth: Google Sheets OAuth2 credentials
   - Consider adding a prior transform/flatten step if you want one row per ledger entry.
22) **Connect Apply Ledger Corrections → Log Fixes to Google Sheets**

23) **Create “Send Notification Email” (Gmail node)**
   - To: `{{ $('Workflow Configuration').first().json.notificationEmail }}`
   - Subject/body: include totals from Apply Ledger Corrections.
   - Auth: Gmail OAuth2 credentials
   - **Important fix:** remove or replace any reference to a non-existent node (see below).
24) **Connect Apply Ledger Corrections → Send Notification Email**

25) **(Optional but implied) Add a “Submit Report to Tax Agent” node**
   - The email template references it, but it is not present.
   - If you add it:
     - likely HTTP Request to `taxAgentApiUrl`
     - output `{ success: true/false }`
   - Then adjust the email expression accordingly, and connect Apply Ledger Corrections → Submit Report to Tax Agent → Send Notification Email (or merge outputs).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The Gmail node references `$('Submit Report to Tax Agent').first().json.success` but there is **no** “Submit Report to Tax Agent” node in the workflow JSON. This will cause an expression/runtime error in the email step unless you add that node or remove the reference. | Fix required for successful email sending |
| The configuration includes `reconciliationThreshold`, but the AI mismatch logic only references it conceptually. To enforce it, pass the threshold explicitly into the tool input (e.g., include it in `financialData` or in the mismatch tool prompt). | Accuracy/consistency improvement |
| Google Sheets “appendOrUpdate” with auto-mapping may not work well with nested structures like `ledgerEntries` arrays. Flatten to one row per mismatch/correction if you want reliable tabular logging. | Data-shaping consideration |
| Sticky note prerequisites mention “NVIDIA API access” and “Input NVIDIA API key for all OpenAI Model nodes”, but the actual workflow uses OpenAI model nodes with OpenAI credentials. Ensure your environment matches your intended provider. | Credential/provider consistency |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.