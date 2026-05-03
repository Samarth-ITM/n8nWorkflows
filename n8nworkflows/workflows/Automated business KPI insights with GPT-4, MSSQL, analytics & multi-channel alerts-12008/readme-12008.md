Automated business KPI insights with GPT-4, MSSQL, analytics & multi-channel alerts

https://n8nworkflows.xyz/workflows/automated-business-kpi-insights-with-gpt-4--mssql--analytics---multi-channel-alerts-12008


# Automated business KPI insights with GPT-4, MSSQL, analytics & multi-channel alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automated business KPI insights with GPT-4, MSSQL, analytics & multi-channel alerts  
**Workflow name (internal):** Business Intelligence copy

This workflow runs daily (scheduled) to collect “yesterday” business metrics from multiple sources (MSSQL, Google Analytics, Google Sheets), validates that each source returned data, aggregates and normalizes the results into a single KPI object, computes performance KPIs (ROAS, CAC), applies business rules to detect risk/opportunity signals, asks an AI agent (OpenAI chat model) to produce a short executive summary, and sends notifications via WhatsApp and Email. If any required data source is missing, it sends an error email and stops further processing for that branch.

### Logical Blocks
1.1 **Scheduled Execution (Entry Point)**  
1.2 **Data Extraction (MSSQL / GA4 / Sheets)**  
1.3 **Data Availability Checks (IF gates + error email)**  
1.4 **Aggregation (Summaries) & Merge**  
1.5 **Normalization & KPI Computation (Code nodes)**  
1.6 **Business Decision Logic (Rules engine in code)**  
1.7 **AI Summary Generation (LangChain Agent + OpenAI model + Calculator tool)**  
1.8 **Notification Delivery (Merge outputs → WhatsApp + Email)**

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled Execution (Entry Point)

**Overview:** Triggers the workflow daily at a fixed hour and fans out into four parallel data collection branches.

**Nodes involved:**  
- Schedule Trigger

**Node details**
- **Schedule Trigger**  
  - **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entrypoint, time-based trigger.  
  - **Config:** Runs on an interval rule with `triggerAtHour: 8` (server timezone).  
  - **Outputs:** Connects to four nodes in parallel: Yesterday's Revenue, Yesterday's Registered Users, Yesterday's Report, Yesterday's Marketing Expenses.  
  - **Edge cases:** Timezone mismatches (n8n instance TZ vs business TZ); missed execution if instance is down; duplicates if schedule changes and workflow is manually executed.

**Sticky note (applies to this area):**  
- **“OVERVIEW”** (architecture and requirements; see Section 5 for captured notes)

---

### 2.2 Data Extraction (MSSQL / GA4 / Sheets)

**Overview:** Pulls raw “yesterday” metrics from database, analytics, and a spreadsheet source.

**Nodes involved:**  
- Yesterday's Revenue (MSSQL)  
- Yesterday's Registered Users (MSSQL)  
- Yesterday's Report (Google Analytics GA4)  
- Yesterday's Marketing Expenses (Google Sheets)

**Node details**
- **Yesterday's Revenue**  
  - **Type / role:** `n8n-nodes-base.microsoftSql` — executes a SQL query to fetch revenue rows.  
  - **Config:** `executeQuery` with `SELECT * from yourtable` (placeholder).  
  - **Credentials:** Microsoft SQL credential required.  
  - **Output:** Goes to **Check data** IF gate.  
  - **Edge cases:** DB auth failures; query returns 0 rows; schema mismatch (no `TotalAmount` field later); long query timeouts.

- **Yesterday's Registered Users**  
  - **Type / role:** `n8n-nodes-base.microsoftSql` — executes a SQL query to fetch registered user rows.  
  - **Config:** `executeQuery` with `select * from yourtable` (placeholder).  
  - **Credentials:** Microsoft SQL credential required.  
  - **Output:** Goes to IF node named **If**.  
  - **Edge cases:** Same as above; later aggregation expects an `email` field for counting.

- **Yesterday's Report**  
  - **Type / role:** `n8n-nodes-base.googleAnalytics` — pulls GA4 report data for yesterday.  
  - **Config:** `dateRange: yesterday`, propertyId set to `420608804`. Metrics/dimensions objects are present but effectively empty (`metricValues` / `dimensionValues` with empty objects), so this likely returns minimal or invalid data depending on node behavior.  
  - **Credentials:** Google Analytics OAuth2 required.  
  - **Output:** Goes to **Condition for google report** IF gate.  
  - **Edge cases:** Missing scopes; property access denied; misconfigured metrics/dimensions causing empty response; API quota.

- **Yesterday's Marketing Expenses**  
  - **Type / role:** `n8n-nodes-base.googleSheets` — reads marketing spend rows from a Google Sheet.  
  - **Config:** Auth via **serviceAccount**; document ID from URL `1ubZBlg0HB0297pqWZyuxpABUpm5PSKPXaD68z53C_MI`, sheet `gid=0`. Operation is not explicitly shown; with v4.7 this typically defaults to a read operation configured by UI (ensure it’s set to “Read” / “Get Many”).  
  - **Credentials:** Google API service account credential required and sheet must be shared with the service account email.  
  - **Output:** Goes to **Condition for marketing reports** IF gate.  
  - **Edge cases:** Sheet permissions; wrong range/columns; “Ad Spend” column missing; numeric values stored as strings.

**Sticky note (covers these nodes):**
- **“DATA EXTRACTION”**  
  1) revenue from DB, 2) new registered users, 3) total users/traffic, 4) marketing expenses, 5) combine into single context.

---

### 2.3 Data Availability Checks (IF gates + error email)

**Overview:** Each data source branch validates existence of returned JSON; if missing, sends an error email. Successful branches proceed to aggregation nodes.

**Nodes involved:**  
- Check data (IF)  
- If (IF)  
- Condition for google report (IF)  
- Condition for marketing reports (IF)  
- Send Error Email (Email)

**Node details**
- **Check data**  
  - **Type / role:** `n8n-nodes-base.if` — validates MSSQL revenue data exists.  
  - **Condition:** Checks if `{{ $('Yesterday\'s Revenue').item.json }}` *exists* (object exists).  
  - **Outputs:**  
    - **true** → Get only Total Revenue Amount  
    - **false** → Send Error Email  
  - **Edge cases:** If query returns empty array, `item.json` still exists, so this check may pass even though there are no rows. Prefer checking a field value or item count.

- **If** (for registered users)  
  - **Type / role:** `n8n-nodes-base.if` — validates registered users data exists.  
  - **Condition:** `{{ $('Yesterday\'s Registered Users').item.json }}` exists.  
  - **Outputs:**  
    - **true** → Get only Total Users (Summarize)  
    - **false** → Send Error Email  
  - **Edge cases:** Same “empty rows still exist” issue.

- **Condition for google report**  
  - **Type / role:** `n8n-nodes-base.if` — validates GA4 report data exists.  
  - **Condition:** `{{ $('Yesterday\'s Report').item.json }}` exists.  
  - **Outputs:**  
    - **true** → Get Only Total Users (Set)  
    - **false** → Send Error Email  
  - **Edge cases:** GA node may return a structured response even when metrics are empty; this check may pass while “Total Users” remains undefined later.

- **Condition for marketing reports**  
  - **Type / role:** `n8n-nodes-base.if` — validates Sheets data exists.  
  - **Condition:** `{{ $('Yesterday\'s Marketing Expenses').item.json }}` exists.  
  - **Outputs:**  
    - **true** → Get only Total Spend Amount  
    - **false** → Send Error Email  
  - **Edge cases:** Same; also beware header mismatch.

- **Send Error Email**  
  - **Type / role:** `n8n-nodes-base.emailSend` — sends failure alert when required input data is missing.  
  - **Config:** SMTP send, plaintext, subject “No Data Returned”, to/from `user@example.com`.  
  - **Inputs:** Triggered by any IF “false” path.  
  - **Edge cases:** SMTP auth errors, blocked sender, rate limits. Also: multiple branches can fail → multiple error emails in the same run unless you add a “merge failures then send once” pattern.

**Sticky note (covers these nodes):**
- **“DATA CHECKER”**  
  Emphasizes blocking execution if any data source is missing and notifying a responsible person.

---

### 2.4 Aggregation (Summaries) & Merge

**Overview:** Aggregates each dataset into a single metric (sum/count/set), then merges the four computed results into one combined stream for normalization.

**Nodes involved:**  
- Get only Total Revenue Amount (Summarize)  
- Get only Total Users (Summarize)  
- Get Only Total Users (Set)  
- Get only Total Spend Amount (Summarize)  
- Merge all result (Merge, 4 inputs)

**Node details**
- **Get only Total Revenue Amount**  
  - **Type / role:** `n8n-nodes-base.summarize` — sums revenue field.  
  - **Config:** Summarize `TotalAmount` with `sum`. Output field becomes `sum_TotalAmount`.  
  - **Input:** From Check data (true).  
  - **Output:** Into Merge all result input 0.  
  - **Edge cases:** Field name mismatch (`TotalAmount` absent or string) → sum becomes 0 or node errors depending on settings.

- **Get only Total Users**  
  - **Type / role:** `n8n-nodes-base.summarize` — counts users based on a field.  
  - **Config:** Field `email` without explicit aggregation set in JSON; in Summarize node it typically defaults to **count** producing `count_email`.  
  - **Input:** From If (true).  
  - **Output:** Into Merge all result input 1.  
  - **Edge cases:** If `email` can be null/empty, counts may differ from actual registrations. Prefer counting rows explicitly.

- **Get Only Total Users** (Set)  
  - **Type / role:** `n8n-nodes-base.set` — maps a value to `Total Users`.  
  - **Config:** Assigns `Total Users` (number) = `{{ $json.totalUsers }}`.  
  - **Input:** From Condition for google report (true).  
  - **Output:** Into Merge all result input 2.  
  - **Edge cases:** Upstream GA output likely doesn’t contain `totalUsers` unless specifically extracted; this may produce `null`/`undefined`.

- **Get only Total Spend Amount**  
  - **Type / role:** `n8n-nodes-base.summarize` — sums marketing spend.  
  - **Config:** Summarize field `Ad Spend` with `sum`, producing `sum_Ad_Spend` (spaces converted to underscore).  
  - **Input:** From Condition for marketing reports (true).  
  - **Output:** Into Merge all result input 3.  
  - **Edge cases:** If column uses different header (e.g., `AdSpend`), sum fails.

- **Merge all result**  
  - **Type / role:** `n8n-nodes-base.merge` — consolidates multiple computed metrics into a single stream.  
  - **Config:** `numberInputs: 4` (explicit 4-way merge).  
  - **Inputs:** From the four aggregation nodes above.  
  - **Output:** Normalize the data.  
  - **Edge cases:** If any successful branch produces multiple items (not truly summarized), merge can output multiple combinations; ensure each branch outputs exactly one item.

**Sticky note (covers merge/normalize context):**
- **“COMBINE REPORTS”**  
  Merge datasets → consolidate into structured JSON → compute KPIs → apply business logic for marketing risk/opportunity.

---

### 2.5 Normalization & KPI Computation (Code)

**Overview:** Converts merged partial items into one consistent JSON object, then computes ROAS and CAC.

**Nodes involved:**  
- Normalize the data (Code)  
- KPI Calculator (Code)

**Node details**
- **Normalize the data**  
  - **Type / role:** `n8n-nodes-base.code` — transforms multiple incoming items into one standardized object.  
  - **Config (logic):** Initializes `{ revenue:0, registeredUsers:0, totalUsers:0, adSpend:0 }`. Iterates through all `items` and picks values based on presence of:  
    - `sum_TotalAmount` → `revenue`  
    - `count_email` → `registeredUsers`  
    - `"Total Users"` → `totalUsers`  
    - `sum_Ad_Spend` → `adSpend`  
    Returns a **single item** with `json: result`.  
  - **Inputs:** From Merge all result.  
  - **Output:** KPI Calculator.  
  - **Edge cases:** If Merge all result produces unexpected item shapes (or multiple items per input), last-seen values win. Also if “Total Users” never set, it stays 0.

- **KPI Calculator**  
  - **Type / role:** `n8n-nodes-base.code` — computes derived KPIs.  
  - **Config (logic):**  
    - `ROAS = revenue / adSpend` if `adSpend > 0` else 0  
    - `CAC = adSpend / users` if `users > 0` else 0 (where `users` = registeredUsers)  
    - Includes placeholders for growth vs yesterday, but commented out (expects `yesterdayRevenue`, `yesterdayUsers` which are not provided by this workflow).  
    - Rounds ROAS/CAC to 2 decimals.  
  - **Input:** Normalize the data output.  
  - **Output:** Business Decision Logic.  
  - **Edge cases:** Division by zero protected; but if revenue/adSpend are strings, `Number()` conversion handles most cases; NaN becomes 0 due to `|| 0`.

**Sticky note (covers this area):**
- **“DATA NORMALIZE”**  
  Consistent types and format, returning totals for revenue, registered users, visitors/total users, and marketing spend.

---

### 2.6 Business Decision Logic (Rules engine)

**Overview:** Applies simple rule thresholds to label the business status and produce a short list of insights used by the AI agent and/or downstream routing.

**Nodes involved:**  
- Business Decision Logic (Code)

**Node details**
- **Business Decision Logic**  
  - **Type / role:** `n8n-nodes-base.code` — assigns status/priority and insight messages.  
  - **Config (logic):**  
    - Default: `status="normal"`, `priority="low"`.  
    - If `ROAS < 1` → `status="risk"`, `priority="high"`, insight “Marketing spend is not generating returns”.  
    - If `ROAS >= 3` → insight “Marketing campaigns are perfoming very well”.  
    - If `CAC > 100` → `status="risk"`, `priority="medium"`, insight “Customer acquision cost is high”.  
    - Outputs `{...kpi, agentStatus, agentPriority, insights}`.  
  - **Outputs / connections:**  
    - Main output index 0 → AI Agent  
    - Main output index 1 → Merge (as the second input for later combination)  
    (This creates a “parallel” path: structured KPI data goes directly to final merge, while also being used for AI generation.)  
  - **Edge cases:** Conflicting thresholds (ROAS < 1 and CAC > 100) will overwrite priority to medium at the end (because CAC rule runs last). If you want “highest” priority, implement explicit ranking logic.

---

### 2.7 AI Summary Generation (LangChain Agent + OpenAI model + tool)

**Overview:** Uses a LangChain-style agent with an OpenAI chat model to generate a short, founder-friendly narrative summary and recommended action. A calculator tool is attached (optional utility).

**Nodes involved:**  
- OpenAI Chat Model  
- Calculator (tool)  
- AI Agent

**Node details**
- **OpenAI Chat Model**  
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM for the agent.  
  - **Config:** Model `gpt-4.1-mini`.  
  - **Credentials:** OpenAI API credential required.  
  - **Output connection:** Connected to AI Agent via `ai_languageModel`.  
  - **Edge cases:** API quota, invalid key, model availability changes, latency/timeouts.

- **Calculator**  
  - **Type / role:** `@n8n/n8n-nodes-langchain.toolCalculator` — tool the agent can call for arithmetic.  
  - **Config:** Default.  
  - **Connection:** Provided to AI Agent via `ai_tool`.  
  - **Edge cases:** Typically minimal; tool use depends on agent prompting.

- **AI Agent**  
  - **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates narrative output from KPI JSON.  
  - **Prompt:**  
    - User text includes:  
      `Business KPIs: {{ JSON.stringify($json) }}`  
      and asks for: (1) one-line summary, (2) one risk, (3) one opportunity, (4) one recommended action.  
    - System message: “AI business agent working for a company founder… short, clear, actionable; no technical terms; do not exaggerate.”  
  - **Inputs:** From Business Decision Logic (main).  
  - **Outputs:** To Merge (combine step). Agent’s output is expected in `$json.output` downstream (as used by WhatsApp/Email nodes).  
  - **Edge cases:** If agent output field naming differs (e.g., `text` instead of `output` depending on node version), downstream expressions break. Pin and test the agent output schema.

**Sticky note (covers this area):**
- **“AI SUMMARY GENERATION”**  
  Explains that the system prompt and model can be changed and that the goal is concise risks/opportunities.

---

### 2.8 Notification Delivery (Merge → WhatsApp + Email)

**Overview:** Combines AI narrative with KPI numbers, then sends the final message via WhatsApp and Email.

**Nodes involved:**  
- Merge (combine)  
- Send message (WhatsApp)  
- Send Final Email (Email)

**Node details**
- **Merge**  
  - **Type / role:** `n8n-nodes-base.merge` — combines AI agent output with KPI JSON.  
  - **Config:** `mode: combine`, `combineBy: combineAll` (attempts to combine all incoming data).  
  - **Inputs:**  
    - Input 0: from AI Agent  
    - Input 1: from Business Decision Logic (second output)  
  - **Outputs:** To Send message and Send Final Email.  
  - **Edge cases:** If one input produces multiple items, combineAll can create unexpected pairing/multiplication. Ensure both sides produce exactly one item.

- **Send message**  
  - **Type / role:** `n8n-nodes-base.whatsApp` — sends WhatsApp message (provider-specific node).  
  - **Config:** Operation `send`, body uses:  
    - `{{ $json.output }}` plus KPI fields `revenue`, `registeredUsers`, `totalUsers`, `adSpend`.  
  - **Credentials:** WhatsApp API credential required (could be Twilio or Meta Cloud API depending on node implementation).  
  - **Edge cases:** Missing recipient configuration (not shown in JSON); WhatsApp template requirements; provider rate limits; `$json.output` missing.

- **Send Final Email**  
  - **Type / role:** `n8n-nodes-base.emailSend` — sends final report email.  
  - **Config:** SMTP send, subject “Yesterday's Revenue Report”, includes AI summary + KPI lines; to/from `user@example.com`.  
  - **Edge cases:** Same as error email; formatting issues if `$json.output` missing.

**Sticky note (covers this area):**
- **“Notification Formatter”**  
  Mentions multi-channel distribution (Email, WhatsApp, optional Slack/Telegram).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Daily trigger / entrypoint | — | Yesterday's Revenue; Yesterday's Registered Users; Yesterday's Report; Yesterday's Marketing Expenses | ## OVERVIEW  / Architecture + Setup & Requirements (MSSQL, GA, Sheets, OpenAI, messaging) |
| Yesterday's Revenue | microsoftSql | Fetch revenue rows from MSSQL | Schedule Trigger | Check data | ## DATA EXTRACTION (revenue, users, analytics, marketing spend; then combine) |
| Check data | if | Gate: revenue data exists | Yesterday's Revenue | Get only Total Revenue Amount (true); Send Error Email (false) | ## DATA CHECKER (IF nodes + email on missing data; block next steps) |
| Get only Total Revenue Amount | summarize | Sum TotalAmount → revenue | Check data (true) | Merge all result | ## COMBINE REPORTS (merge datasets; consolidate; KPIs; business rules) |
| Yesterday's Registered Users | microsoftSql | Fetch registered users rows from MSSQL | Schedule Trigger | If | ## DATA EXTRACTION (revenue, users, analytics, marketing spend; then combine) |
| If | if | Gate: registered users data exists | Yesterday's Registered Users | Get only Total Users (true); Send Error Email (false) | ## DATA CHECKER (IF nodes + email on missing data; block next steps) |
| Get only Total Users | summarize | Count email → registeredUsers | If (true) | Merge all result | ## COMBINE REPORTS (merge datasets; consolidate; KPIs; business rules) |
| Yesterday's Report | googleAnalytics | Fetch GA4 report (yesterday) | Schedule Trigger | Condition for google report | ## DATA EXTRACTION (revenue, users, analytics, marketing spend; then combine) |
| Condition for google report | if | Gate: GA report exists | Yesterday's Report | Get Only Total Users (true); Send Error Email (false) | ## DATA CHECKER (IF nodes + email on missing data; block next steps) |
| Get Only Total Users | set | Map $json.totalUsers → “Total Users” | Condition for google report (true) | Merge all result | ## COMBINE REPORTS (merge datasets; consolidate; KPIs; business rules) |
| Yesterday's Marketing Expenses | googleSheets | Read ad spend rows from Google Sheets | Schedule Trigger | Condition for marketing reports | ## DATA EXTRACTION (revenue, users, analytics, marketing spend; then combine) |
| Condition for marketing reports | if | Gate: marketing spend data exists | Yesterday's Marketing Expenses | Get only Total Spend Amount (true); Send Error Email (false) | ## DATA CHECKER (IF nodes + email on missing data; block next steps) |
| Get only Total Spend Amount | summarize | Sum “Ad Spend” → adSpend | Condition for marketing reports (true) | Merge all result | ## COMBINE REPORTS (merge datasets; consolidate; KPIs; business rules) |
| Merge all result | merge | Merge 4 aggregated metrics | Get only Total Revenue Amount; Get only Total Users; Get Only Total Users; Get only Total Spend Amount | Normalize the data | ## COMBINE REPORTS (merge datasets; consolidate; KPIs; business rules) |
| Normalize the data | code | Produce one normalized KPI object | Merge all result | KPI Calculator | ## DATA NORMALIZE (consistent types + totals for revenue/users/visits/spend) |
| KPI Calculator | code | Compute ROAS, CAC | Normalize the data | Business Decision Logic | ## COMBINE REPORTS (merge datasets; consolidate; KPIs; business rules) |
| Business Decision Logic | code | Tag status/priority + insights | KPI Calculator | AI Agent; Merge | ## COMBINE REPORTS (merge datasets; consolidate; KPIs; business rules) |
| OpenAI Chat Model | lmChatOpenAi | LLM provider for agent | — | AI Agent (ai_languageModel) | ## AI SUMMARY GENERATION (founder-friendly summary; model/prompt configurable) |
| Calculator | toolCalculator | Agent tool for arithmetic | — | AI Agent (ai_tool) | ## AI SUMMARY GENERATION (founder-friendly summary; model/prompt configurable) |
| AI Agent | langchain.agent | Generate executive summary text | Business Decision Logic | Merge | ## AI SUMMARY GENERATION (founder-friendly summary; model/prompt configurable) |
| Merge | merge | Combine AI output + KPI JSON | AI Agent; Business Decision Logic | Send message; Send Final Email | ## Notification Formatter (prepare message; deliver via WhatsApp/Email/Slack/Telegram) |
| Send message | whatsApp | Send WhatsApp alert | Merge | — | ## Notification Formatter (prepare message; deliver via WhatsApp/Email/Slack/Telegram) |
| Send Final Email | emailSend | Send final report email | Merge | — | ## Notification Formatter (prepare message; deliver via WhatsApp/Email/Slack/Telegram) |
| Send Error Email | emailSend | Alert on missing data | Check data (false); If (false); Condition for google report (false); Condition for marketing reports (false) | — | ## DATA CHECKER (IF nodes + email on missing data; block next steps) |
| Sticky Note | stickyNote | Commentary | — | — | ## OVERVIEW (content block) |
| Sticky Note3 | stickyNote | Commentary | — | — | ## Notification Formatter (content block) |
| Sticky Note4 | stickyNote | Commentary | — | — | ## DATA EXTRACTION (content block) |
| Sticky Note5 | stickyNote | Commentary | — | — | ## DATA CHECKER (content block) |
| Sticky Note6 | stickyNote | Commentary | — | — | ## DATA NORMALIZE (content block) |
| Sticky Note7 | stickyNote | Commentary | — | — | ## COMBINE REPORTS (content block) |
| Sticky Note8 | stickyNote | Commentary | — | — | ## AI SUMMARY GENERATION (content block) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it (e.g.) “Automated business KPI insights with GPT-4, MSSQL, analytics & multi-channel alerts”.

2) **Add Trigger**
- Add **Schedule Trigger**  
  - Set it to run **daily at 08:00** (confirm timezone in n8n settings).

3) **Add data extraction nodes (4 parallel branches)**
- Add **Microsoft SQL** node “Yesterday's Revenue”  
  - Operation: **Execute Query**  
  - Query: replace placeholder with a real query that returns a `TotalAmount` column (or adjust later nodes):  
    - Example: `SELECT TotalAmount FROM Orders WHERE OrderDate = CAST(GETDATE()-1 AS date);`
  - Configure MSSQL credentials (host, db, user/password or integrated auth).

- Add **Microsoft SQL** node “Yesterday's Registered Users”  
  - Operation: **Execute Query**  
  - Query: ensure it returns a column you can count (here it expects `email`):  
    - Example: `SELECT email FROM Users WHERE CAST(CreatedAt AS date)=CAST(GETDATE()-1 AS date);`

- Add **Google Analytics (GA4)** node “Yesterday's Report”  
  - Date range: **yesterday**  
  - Property ID: select your GA4 property  
  - Configure at least one **metric** and (optionally) a **dimension** so the response contains the value you need (e.g., activeUsers / totalUsers).  
  - Set Google Analytics OAuth2 credentials (scopes for GA4 reporting).

- Add **Google Sheets** node “Yesterday's Marketing Expenses”  
  - Authentication: **Service Account**  
  - Document: select spreadsheet `Yesterday's Data`  
  - Sheet: `gid=0` (or pick the relevant tab)  
  - Configure read operation/range so output includes a numeric column named exactly **Ad Spend** (or adjust summarize field name).  
  - Ensure the sheet is shared with the service account email.

4) **Connect Schedule Trigger to all 4 extraction nodes** (fan-out)
- Drag connections from Schedule Trigger to each of the four nodes.

5) **Add IF gates (one per branch)**
- Add **IF** node “Check data” after “Yesterday's Revenue”  
  - Condition: **Exists** check on the incoming data (better: check an actual field or item count).  
  - Connect: Revenue → Check data.

- Add **IF** node “If” after “Yesterday's Registered Users”  
  - Similar exists check.  
  - Connect: Registered Users → If.

- Add **IF** node “Condition for google report” after “Yesterday's Report”  
  - Similar exists check.  
  - Connect: GA node → Condition for google report.

- Add **IF** node “Condition for marketing reports” after “Yesterday's Marketing Expenses”  
  - Similar exists check.  
  - Connect: Sheets node → Condition for marketing reports.

6) **Add the shared error email node**
- Add **Email Send** node “Send Error Email”  
  - SMTP credentials (host/port/user/pass; from address).  
  - To: responsible person  
  - Subject: “No Data Returned”  
  - Body: the failure text.
- Connect each IF node’s **false** output to **Send Error Email**.

7) **Add aggregation nodes on the “true” paths**
- After **Check data (true)** add **Summarize** “Get only Total Revenue Amount”  
  - Summarize field: `TotalAmount`, aggregation: **sum**.

- After **If (true)** add **Summarize** “Get only Total Users”  
  - Count `email` (or count rows, depending on your data design).

- After **Condition for google report (true)** add **Set** “Get Only Total Users”  
  - Create field `Total Users` (number).  
  - Map it from the GA output (you may need an intermediate Set/Code node to extract the metric properly). The provided workflow uses `{{$json.totalUsers}}`.

- After **Condition for marketing reports (true)** add **Summarize** “Get only Total Spend Amount”  
  - Summarize field: `Ad Spend`, aggregation: **sum**.

8) **Merge the four aggregated results**
- Add **Merge** node “Merge all result”  
  - Set **Number of inputs = 4**.  
- Connect:  
  - Revenue summarize → Merge all result input 0  
  - Registered users summarize → input 1  
  - Total users set → input 2  
  - Spend summarize → input 3

9) **Normalize**
- Add **Code** node “Normalize the data”  
  - Paste logic equivalent to: collect `sum_TotalAmount`, `count_email`, `"Total Users"`, `sum_Ad_Spend` into `{revenue, registeredUsers, totalUsers, adSpend}` and return one item.

10) **Compute KPIs**
- Add **Code** node “KPI Calculator”  
  - Compute `ROAS` and `CAC` from normalized fields, rounding to 2 decimals.

11) **Business rules**
- Add **Code** node “Business Decision Logic”  
  - Implement thresholds for ROAS and CAC; output `agentStatus`, `agentPriority`, and `insights[]`.

12) **AI Agent setup**
- Add **OpenAI Chat Model** node  
  - Select model (e.g., `gpt-4.1-mini`).  
  - Configure OpenAI API credentials.

- Add **AI Agent** node  
  - System message: founder-friendly, short, non-technical, non-exaggerated.  
  - User prompt: include KPI JSON and ask for summary/risk/opportunity/action.  
  - Connect **OpenAI Chat Model → AI Agent** via the agent’s **language model** input.

- Add **Calculator tool** node (optional but included in original)  
  - Connect it to AI Agent as a **tool**.

- Connect **Business Decision Logic → AI Agent** (main).

13) **Combine AI output with KPI data**
- Add **Merge** node “Merge”  
  - Mode: **Combine**  
  - Combine by: **Combine All**  
- Connect:  
  - AI Agent → Merge input 0  
  - Business Decision Logic → Merge input 1  
  (In the provided workflow, Business Decision Logic sends to Merge via a second output connection; in practice you can just branch the same main output to both AI Agent and Merge.)

14) **Notifications**
- Add **WhatsApp** node “Send message”  
  - Configure WhatsApp credentials/provider.  
  - Message body uses the AI output field and KPI fields. Ensure the AI Agent output field name matches your expression (often `{{$json.output}}` or `{{$json.text}}` depending on node version).

- Add **Email Send** node “Send Final Email”  
  - SMTP credentials.  
  - Subject: “Yesterday's Revenue Report”.  
  - Body: include AI summary and KPI numbers.

- Connect **Merge → Send message** and **Merge → Send Final Email**.

15) **Test with manual execution**
- Run each extraction node to confirm it returns expected fields (`TotalAmount`, `email`, GA metric you map to Total Users, `Ad Spend`).  
- Validate Summarize outputs (`sum_TotalAmount`, `count_email`, `sum_Ad_Spend`).  
- Confirm AI Agent output contains the field referenced by notification nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow is an AI-powered business intelligence agent designed for founders and business owners… collects key business metrics… calculates KPIs… uses AI reasoning… sends actionable notifications.” | Sticky note: OVERVIEW |
| Architecture: Scheduler → Data Collection (MSSQL, Analytics, Sheets) → Merge → Data Consolidation → KPI Calculation → Business Decision Logic → AI Reasoning Agent → Notification Formatter → Smart Alerts | Sticky note: OVERVIEW |
| Setup & Requirements: API access to MSSQL / GA / Sheets; OpenAI or Gemini; messaging via Gmail/SMTP, Twilio WhatsApp, Slack/Telegram | Sticky note: OVERVIEW |
| Google Analytics console link | https://analytics.google.com/analytics/web/#/ |
| Google Sheet used for marketing expenses | https://docs.google.com/spreadsheets/d/1ubZBlg0HB0297pqWZyuxpABUpm5PSKPXaD68z53C_MI |
| Notification formatter guidance: deliver via Email (SMTP), WhatsApp (Twilio/Cloud API), optional Slack/Telegram | Sticky note: Notification Formatter |
| AI summary note: system prompt/model can be changed; goal is concise summary, risks, opportunities | Sticky note: AI SUMMARY GENERATION |

