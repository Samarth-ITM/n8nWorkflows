Generate AI inventory reorder recommendations from Airtable with GPT-4o, Slack and email

https://n8nworkflows.xyz/workflows/generate-ai-inventory-reorder-recommendations-from-airtable-with-gpt-4o--slack-and-email-11294


# Generate AI inventory reorder recommendations from Airtable with GPT-4o, Slack and email

## 1. Workflow Overview

**Purpose:**  
This workflow pulls inventory rows from Airtable, validates them, uses Azure OpenAI GPT‑4o to compute **reorder point**, **safety stock**, and **stock status** using strict formulas, then distributes operational summaries via **Slack** and **Gmail**. It also logs invalid input rows to **Google Sheets**, logs optimization results to **Notion**, and creates an **Asana** task for follow-up. A global **error trigger** emails an alert if the workflow fails.

**Target use cases:**
- Daily/weekly inventory health checks and reorder planning
- Automated ops communication (Slack + email)
- Auditability (invalid row logging, Notion logging, Asana tasks)
- Safe AI calculations with controlled JSON output

### 1.1 Entry & Data Intake
Manual execution starts the workflow and fetches inventory records from Airtable.

### 1.2 Validation & Audit Logging
Each record is checked for a required structure (currently only `id` is enforced). Invalid rows are appended to a Google Sheet.

### 1.3 AI Optimization (Strict Formulas)
GPT‑4o (Azure OpenAI) is used as an “inventory optimization engine” to compute:
- `SuggestedReorderPoint = ReorderLevel * 1.2`
- `SuggestedSafetyStock = ReorderLevel * 0.5`
- `StockStatus` based on thresholds  
Outputs are expected as raw JSON only. Results are consolidated via a Code node.

### 1.4 Multi-Channel Summaries & Actions
From the merged AI results:
- GPT‑4o generates a Slack-ready summary → Slack node posts/sends it.
- GPT‑4o generates an email-ready summary → Gmail sends it.
- Notion logs the optimization dataset (currently in a simplistic way).
- Asana creates a task (based on the first result item only).

### 1.5 Error Handling
Any node error triggers an error workflow path that emails an “Workflow Error Alert”.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Entry & Airtable Fetch
**Overview:** Starts on manual execution and pulls inventory records from a specific Airtable base/table using a Search operation.  
**Nodes involved:**  
- When clicking ‘Execute workflow’  
- Fetch Inventory Records from Airtable

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point for manual runs.
- **Configuration:** No parameters.
- **Connections:**  
  - **Output →** Fetch Inventory Records from Airtable
- **Failure modes:** None (only user-triggered).

#### Node: Fetch Inventory Records from Airtable
- **Type / role:** Airtable node (`n8n-nodes-base.airtable`) — reads inventory rows.
- **Configuration (interpreted):**
  - **Operation:** `search`
  - **Base:** “Lead Manager” (ID `appxjEpOgye5YQG1J`)
  - **Table:** “Leads” (ID `tbl4TiSSi2uUJaUOO`)
  - **Options:** default/empty (no filter formula shown)
- **Credentials:** Airtable Personal Access Token (`airtableTokenApi`)
- **Connections:**  
  - **Input ←** Manual Trigger  
  - **Output →** Validate Inventory Record Structure
- **Edge cases / failure types:**
  - Auth/permission failures (token scope/base access)
  - Large result sets causing long runtime or memory pressure
  - “Search” without a filter may return more records than expected
  - Schema mismatch (fields expected by later AI prompts may not exist)

---

### Block 2.2 — Validation & Invalid-Row Audit Logging
**Overview:** Routes records based on minimal validation. “Valid” records go to AI optimization; invalid ones are appended to Google Sheets for audit.  
**Nodes involved:**  
- Validate Inventory Record Structure  
- Log Invalid Inventory Rows to Google Sheet

#### Node: Validate Inventory Record Structure
- **Type / role:** IF node (`n8n-nodes-base.if`) — basic validation gate.
- **Configuration (interpreted):**
  - Condition: `$json.id` **is not empty**
  - Strict validation enabled at the IF node level (`typeValidation: strict`)
  - **Important:** This validates only the Airtable record `id`, not business fields like `ItemName`, `SKU`, `QuantityInStock`, `ReorderLevel`.
- **Connections:**  
  - **Input ←** Fetch Inventory Records from Airtable  
  - **True/Output 0 →** Generate Inventory Optimization Output (AI)  
  - **False/Output 1 →** Log Invalid Inventory Rows to Google Sheet
- **Edge cases / failure types:**
  - Records missing required business fields will still pass if `id` exists (likely), potentially causing AI errors or nonsensical calculations.
  - If Airtable returns items in a different shape than expected, expression evaluation might fail (rare here, since `$json.id` typically exists).

#### Node: Log Invalid Inventory Rows to Google Sheet
- **Type / role:** Google Sheets node (`n8n-nodes-base.googleSheets`) — appends audit rows.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Spreadsheet:** `sample_leads_50` (ID `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`)
  - **Sheet tab:** `hacker news` (gid `1318002027`)
  - **Column mapping:** Defines one column `output` (but mapping content is not explicitly set in the provided config)
- **Credentials:** Google Sheets OAuth2 (`automations@techdome.ai`)
- **Connections:**  
  - **Input ←** Validate Inventory Record Structure (false path)
- **Edge cases / failure types:**
  - Append may fail if the sheet structure/permissions change
  - Since no explicit field mapping is shown, it may append blank `output` unless n8n auto-maps (often it won’t without mapping)
  - Quota limits, rate limiting, or expired OAuth tokens

---

### Block 2.3 — AI Optimization Engine (Strict JSON + Consolidation)
**Overview:** Uses GPT‑4o (Azure) to calculate inventory metrics using fixed formulas, enforces “raw JSON only”, then merges potentially multiple AI outputs into a single consolidated `results[]` array.  
**Nodes involved:**  
- Configure GPT-4o — Inventory Optimization Model  
- Generate Inventory Optimization Output (AI)  
- Merge AI Optimization Results

#### Node: Configure GPT-4o — Inventory Optimization Model
- **Type / role:** Azure OpenAI Chat Model (LangChain) (`@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`) — provides the model to the Agent node via `ai_languageModel`.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Options: default
- **Credentials:** Azure OpenAI account
- **Connections:**  
  - **ai_languageModel →** Generate Inventory Optimization Output (AI)
- **Edge cases / failure types:**
  - Azure deployment/model mismatch (your Azure deployment must support “gpt-4o” as configured in n8n credential settings)
  - Network/timeouts; token limits if large Airtable payloads are sent

#### Node: Generate Inventory Optimization Output (AI)
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — prompts GPT to produce strict JSON calculations.
- **Configuration (interpreted):**
  - **Prompt includes input:** `{{ JSON.stringify($json, null, 2) }}`
  - **Hard constraints:** “Return ONLY raw JSON”, no markdown/backticks
  - **Formulas enforced:**
    - `SuggestedReorderPoint = ReorderLevel * 1.2`
    - `SuggestedSafetyStock = ReorderLevel * 0.5`
  - **StockStatus rules:**
    - `Critical` if `QuantityInStock <= SuggestedSafetyStock`
    - `Needs Reorder` if `QuantityInStock <= SuggestedReorderPoint`
    - `OK` otherwise
  - **Output parser enabled:** `hasOutputParser: true` (meaning n8n will try to parse/structure the output)
- **Model binding:** Receives model from “Configure GPT-4o — Inventory Optimization Model” via `ai_languageModel`.
- **Connections:**  
  - **Input ←** Validate Inventory Record Structure (true path)  
  - **Output →** Merge AI Optimization Results
- **Edge cases / failure types:**
  - If `QuantityInStock` or `ReorderLevel` are missing/non-numeric strings, the model may output invalid JSON or wrong types (despite instructions).
  - Very large inventory payloads may exceed context/token limits; results could be partial.
  - Even with “no markdown” instructions, model may still emit backticks; workflow anticipates this in the merge Code node.

#### Node: Merge AI Optimization Results
- **Type / role:** Code node (`n8n-nodes-base.code`) — consolidates multiple AI outputs into one object.
- **Configuration (interpreted):**
  - Reads all incoming items: `let items = $input.all();`
  - Extracts output from `item.json.output || item.json`
  - Strips markdown fences: `replace(/```json|```/g, '')`
  - Parses JSON; if `parsed.results` exists, appends to `mergedResults`
  - Returns a single item: `{ results: mergedResults }`
- **Connections:**  
  - **Input ←** Generate Inventory Optimization Output (AI)  
  - **Output →** (fan-out)
    - Generate Inventory Email Summary
    - Generate Inventory Slack Summary
    - Create Asana Task for Inventory Db
    - Log Trade Decision to Notion (inventory db)
- **Edge cases / failure types:**
  - Silent failure by design: invalid JSON is skipped with no error raised (could hide model issues).
  - If the agent node already returns structured JSON (not a string), `String(out)` may produce `"[object Object]"` which fails JSON.parse and results get dropped. (This is a key integration risk given `hasOutputParser: true`.)
  - If all parses fail, downstream nodes receive `{ results: [] }`, which can break Asana/Notion expressions that assume `results[0]` exists.

---

### Block 2.4 — Slack Summary Generation & Delivery
**Overview:** Generates a Slack-formatted message using GPT‑4o and sends it via Slack API.  
**Nodes involved:**  
- Configure GPT-4o Model1  
- Generate Inventory Slack Summary  
- Notify Operations Team on Slack

#### Node: Configure GPT-4o Model1
- **Type / role:** Azure OpenAI Chat Model (LangChain) — Slack summary model provider.
- **Configuration:** Model `gpt-4o`, default options.
- **Credentials:** Azure OpenAI account
- **Connections:**  
  - **ai_languageModel →** Generate Inventory Slack Summary
- **Edge cases:** Same as other Azure model node (deployment mismatch, auth, timeouts).

#### Node: Generate Inventory Slack Summary
- **Type / role:** LangChain Agent — produces Slack-friendly condensed status with emojis.
- **Configuration (interpreted):**
  - Input: merged optimization JSON via `{{ JSON.stringify($json, null, 2) }}`
  - Requirements:
    - Overall one-line health
    - Compact bullets per item
    - Highlight below reorder / below safety
    - Emojis (⚠️ 🔴 🟢)
    - End with one recommended action line
- **Model binding:** from “Configure GPT-4o Model1”
- **Connections:**  
  - **Input ←** Merge AI Optimization Results  
  - **Output →** Notify Operations Team on Slack
- **Edge cases / failure types:**
  - Output format not guaranteed; Slack node expects `$json.output` later
  - If results array is empty, agent should still produce a message, but prompt doesn’t explicitly force “no issues detected” like email prompt does.

#### Node: Notify Operations Team on Slack
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — sends message.
- **Configuration (interpreted):**
  - Sends `text: {{ $json.output }}`
  - **Target:** configured as `select: user` with user `U09HMPVD466` (“newscctv22”)
  - `includeLinkToWorkflow: false`
- **Credentials:** Slack API credential (“Slack account vivek”)
- **Sticky note mismatch:** Node note says it sends to “#product-faqs channel”, but config targets a **user**, not a channel.
- **Connections:**  
  - **Input ←** Generate Inventory Slack Summary
- **Edge cases / failure types:**
  - If `$json.output` missing (agent returns different field), message will be blank or node fails.
  - Slack permission scopes (chat:write, im:write) and DM availability.
  - If you intended a channel, you must switch to channel mode.

---

### Block 2.5 — Email Summary Generation & Delivery
**Overview:** Creates an operations email summary using GPT‑4o and sends it via Gmail.  
**Nodes involved:**  
- Configure GPT-4o Model  
- Generate Inventory Email Summary  
- Email Inventory Summary to Manager

#### Node: Configure GPT-4o Model
- **Type / role:** Azure OpenAI Chat Model (LangChain) — email summary model provider.
- **Configuration:** Model `gpt-4o`, default options.
- **Credentials:** Azure OpenAI account
- **Connections:**  
  - **ai_languageModel →** Generate Inventory Email Summary
- **Edge cases:** Same Azure model risks (deployment mismatch, auth).

#### Node: Generate Inventory Email Summary
- **Type / role:** LangChain Agent — produces email subject/body in business format.
- **Configuration (interpreted):**
  - Input: merged optimization JSON
  - Must include: subject, overview, item bullets, alerts, recommended actions, closing
  - System message explicitly says: don’t change numbers; no fictional items; if all healthy, still say “No issues detected”.
- **Connections:**  
  - **Input ←** Merge AI Optimization Results  
  - **Output →** Email Inventory Summary to Manager
- **Edge cases / failure types:**
  - Output is free-form text; Gmail node uses `$json.output`. If agent outputs another property name, email body may be empty.
  - If results are empty, prompt requires still producing a valid “No issues detected” email (good safeguard).

#### Node: Email Inventory Summary to Manager
- **Type / role:** Gmail node (`n8n-nodes-base.gmail`) — sends email.
- **Configuration (interpreted):**
  - **To:** `=` (appears empty/invalid; likely misconfigured)
  - **Subject:** `"Consent Manager Governance"` (does not match inventory context; likely placeholder)
  - **Message body:** `{{ $json.output }}`
- **Credentials:** Gmail OAuth2 (“Gmail account”)
- **Connections:**  
  - **Input ←** Generate Inventory Email Summary
- **Edge cases / failure types:**
  - Will fail if **SendTo** is blank or invalid.
  - Subject mismatch may confuse recipients.
  - Gmail OAuth token expiration or insufficient scopes.

---

### Block 2.6 — Logging & Task Creation (Notion + Asana)
**Overview:** Persists optimization results for traceability (Notion) and creates an actionable task (Asana).  
**Nodes involved:**  
- Log Trade Decision to Notion (inventory db)  
- Create Asana Task for Inventory Db

#### Node: Log Trade Decision to Notion (inventory db)
- **Type / role:** Notion node (`n8n-nodes-base.notion`) — creates a database page.
- **Configuration (interpreted):**
  - Resource: `databasePage` (create a new page in a database)
  - Database: “inventory reorder”
  - Property mapping:
    - Sets `Name` (title) to `{{ $json.results }}`
- **Connections:**  
  - **Input ←** Merge AI Optimization Results
- **Edge cases / failure types:**
  - `Name` is a title property; assigning an **array/object** (`results`) may be rejected or coerced poorly. Typically you’d set something like a date + summary string, or iterate items and create one page per item.
  - Database schema mismatch: property name `Name|title` must exist exactly.
  - Notion rate limits, permissions, integration access to database.

#### Node: Create Asana Task for Inventory Db
- **Type / role:** Asana node (`n8n-nodes-base.asana`) — creates a task for follow-up.
- **Configuration (interpreted):**
  - Task name built from **first result only**: `$json.results[0]...`
  - Notes: multiline inventory alert template using first result fields
  - Due date: tomorrow: `{{ $now.plus({ days: 1 }).toISODate() }}`
  - Workspace: `1212551193156936`
  - Project: `1212565062132347`
  - Auth: OAuth2
- **Connections:**  
  - **Input ←** Merge AI Optimization Results
- **Edge cases / failure types:**
  - If `results` is empty, `$json.results[0]` is `undefined` → expressions will error and the node will fail.
  - Only one task is created even if many SKUs are critical; may not meet operational needs.
  - Asana permissions/project access, rate limiting.

---

### Block 2.7 — Global Error Handling
**Overview:** On any workflow failure, an error trigger fires and sends a Gmail alert with node name, error message, and timestamp.  
**Nodes involved:**  
- Workflow Error Handler  
- Send a message1

#### Node: Workflow Error Handler
- **Type / role:** Error Trigger (`n8n-nodes-base.errorTrigger`) — secondary entry point when workflow errors.
- **Configuration:** none.
- **Connections:**  
  - **Output →** Send a message1
- **Edge cases:** Only triggers on failures in the workflow; if Gmail credential itself is broken, error alert may fail.

#### Node: Send a message1
- **Type / role:** Gmail node — sends error alert.
- **Configuration (interpreted):**
  - Email type: text
  - Subject: “Workflow Error Alert”
  - Message includes:
    - Error node name: `{{ $json.node.name }}`
    - Error message: `{{ $json.error.message }}`
    - Timestamp: `{{ $now.toISO() }}`
- **Credentials:** Gmail OAuth2 (“Gmail credentials”)
- **Connections:**  
  - **Input ←** Workflow Error Handler
- **Edge cases:**
  - Missing “sendTo” parameter is not shown; if unset, alert may fail.
  - If `$json.error.message` absent for some failure types, message may be incomplete.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Fetch Inventory Records from Airtable | ## 📦 Automate Inventory Reordering from Airtable using GPT-4o, Slack & Email … |
| Fetch Inventory Records from Airtable | Airtable | Load inventory rows | When clicking ‘Execute workflow’ | Validate Inventory Record Structure | ## Intake & Validation Block … |
| Validate Inventory Record Structure | IF | Validate record has required fields | Fetch Inventory Records from Airtable | Generate Inventory Optimization Output (AI); Log Invalid Inventory Rows to Google Sheet | ## Intake & Validation Block … |
| Log Invalid Inventory Rows to Google Sheet | Google Sheets | Audit log invalid rows | Validate Inventory Record Structure (false) | — | ## Intake & Validation Block … |
| Configure GPT-4o — Inventory Optimization Model | Azure OpenAI Chat Model (LangChain) | Provides GPT‑4o model to optimization agent | — | Generate Inventory Optimization Output (AI) (ai_languageModel) | ## AI Optimization Engine … |
| Generate Inventory Optimization Output (AI) | LangChain Agent | Compute reorder point/safety stock/status as strict JSON | Validate Inventory Record Structure (true) + ai_languageModel | Merge AI Optimization Results | ## AI Optimization Engine … |
| Merge AI Optimization Results  | Code | Parse/clean/merge AI JSON outputs | Generate Inventory Optimization Output (AI) | Generate Inventory Email Summary ; Generate Inventory Slack Summary; Create Asana Task for Inventory Db; Log Trade Decision to Notion (inventory db) | ## AI Optimization Engine … |
| Configure GPT-4o Model1 | Azure OpenAI Chat Model (LangChain) | Provides GPT‑4o model to Slack summary agent | — | Generate Inventory Slack Summary (ai_languageModel) | ## Slack Summary Generation … |
| Generate Inventory Slack Summary | LangChain Agent | Create Slack-friendly summary | Merge AI Optimization Results  + ai_languageModel | Notify Operations Team on Slack | ## Slack Summary Generation … |
| Notify Operations Team on Slack | Slack | Send Slack message | Generate Inventory Slack Summary | — | ## Slack Summary Generation … |
| Configure GPT-4o Model | Azure OpenAI Chat Model (LangChain) | Provides GPT‑4o model to email summary agent | — | Generate Inventory Email Summary  (ai_languageModel) | ## Email Summary Generation … |
| Generate Inventory Email Summary  | LangChain Agent | Create email subject/body summary | Merge AI Optimization Results  + ai_languageModel | Email Inventory Summary to Manager | ## Email Summary Generation … |
| Email Inventory Summary to Manager | Gmail | Send email to manager | Generate Inventory Email Summary  | — | ## Email Summary Generation … |
| Log Trade Decision to Notion (inventory db) | Notion | Persist results to Notion database | Merge AI Optimization Results  | — | ## 🗂️ Task & Record Logging … |
| Create Asana Task for Inventory Db | Asana | Create ops follow-up task | Merge AI Optimization Results  | — | ## 🗂️ Task & Record Logging … |
| Workflow Error Handler | Error Trigger | Catch workflow errors | — | Send a message1 | ## Error Handling… |
| Send a message1 | Gmail | Email error alert | Workflow Error Handler | — | ## Error Handling… |
| Sticky Note | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note1 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note5 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note8 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note11 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note7 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note2 | Sticky Note | Documentation/comment | — | — |  |
| Sticky Note3 | Sticky Note | Documentation/comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: *Automate Inventory Reordering from Airtable using GPT-4o, Slack & Email*
- (Optional) Add sticky notes with the same text blocks for maintainability.

2) **Add Trigger**
- Add node: **Manual Trigger**
- Name: *When clicking ‘Execute workflow’*

3) **Add Airtable fetch**
- Add node: **Airtable**
- Name: *Fetch Inventory Records from Airtable*
- Credentials: create/select **Airtable Personal Access Token**
- Operation: **Search**
- Select **Base** (your inventory base)
- Select **Table** (your inventory table)
- (Recommended) Add a filter formula or view to limit rows (optional but advisable).
- Connect: Manual Trigger → Airtable

4) **Add validation gate**
- Add node: **IF**
- Name: *Validate Inventory Record Structure*
- Condition: **String** → `={{ $json.id }}` → **is not empty**
- Connect: Airtable → IF
- Keep two outputs: **true** (valid) and **false** (invalid).

5) **Add Google Sheets invalid-row logging**
- Add node: **Google Sheets**
- Name: *Log Invalid Inventory Rows to Google Sheet*
- Credentials: **Google Sheets OAuth2**
- Operation: **Append**
- Choose Spreadsheet + Sheet tab for audit log
- Create a column mapping such as:
  - `output` = `={{ JSON.stringify($json) }}`
- Connect: IF (false) → Google Sheets

6) **Add Azure OpenAI model for optimization**
- Add node: **Azure OpenAI Chat Model (LangChain)** (n8n LangChain)
- Name: *Configure GPT-4o — Inventory Optimization Model*
- Model: `gpt-4o`
- Credentials: **Azure OpenAI** (must have a deployment compatible with GPT‑4o in your Azure setup)

7) **Add AI optimization agent**
- Add node: **AI Agent (LangChain)**
- Name: *Generate Inventory Optimization Output (AI)*
- Prompt: include your input JSON and demand strict output:
  - Include `{{ JSON.stringify($json, null, 2) }}`
  - Force **raw JSON only**, no markdown/backticks
  - Include formulas and StockStatus rules exactly
- Enable/keep output parsing if you plan to handle structured output consistently.
- Connect:
  - IF (true) → AI Optimization Agent (main)
  - Azure OpenAI model node → AI Agent (ai_languageModel connection)

8) **Add merge/consolidation Code node**
- Add node: **Code**
- Name: *Merge AI Optimization Results*
- Paste logic equivalent to:
  - iterate `$input.all()`
  - extract `item.json.output || item.json`
  - strip backticks
  - JSON.parse and collect `results`
  - output `{ results: mergedResults }`
- Connect: AI Optimization Agent → Code node

9) **Add Slack summary model + agent**
- Add node: **Azure OpenAI Chat Model (LangChain)**
- Name: *Configure GPT-4o Model1*
- Model: `gpt-4o`
- Add node: **AI Agent (LangChain)**
- Name: *Generate Inventory Slack Summary*
- Prompt: Slack-friendly summary, include emojis, compact list, end with one action.
- Connect:
  - Merge Code → Slack Summary Agent (main)
  - Configure GPT-4o Model1 → Slack Summary Agent (ai_languageModel)

10) **Add Slack send**
- Add node: **Slack**
- Name: *Notify Operations Team on Slack*
- Credentials: **Slack API**
- Set destination:
  - Either **Channel** (recommended for ops) or **User** (DM)
- Text: `={{ $json.output }}`
- Connect: Slack Summary Agent → Slack node

11) **Add Email summary model + agent**
- Add node: **Azure OpenAI Chat Model (LangChain)**
- Name: *Configure GPT-4o Model*
- Model: `gpt-4o`
- Add node: **AI Agent (LangChain)**
- Name: *Generate Inventory Email Summary*
- Prompt: must include subject + structured email sections; do not alter numbers; if no issues, state “No issues detected”.
- Connect:
  - Merge Code → Email Summary Agent (main)
  - Configure GPT-4o Model → Email Summary Agent (ai_languageModel)

12) **Add Gmail send**
- Add node: **Gmail**
- Name: *Email Inventory Summary to Manager*
- Credentials: **Gmail OAuth2**
- To: set an actual email address (e.g., `ops-manager@company.com`)
- Subject: set an inventory-appropriate subject or map from AI output (if you choose)
- Message: `={{ $json.output }}`
- Connect: Email Summary Agent → Gmail node

13) **Add Notion logging**
- Add node: **Notion**
- Name: *Log Trade Decision to Notion (inventory db)*
- Credentials: **Notion API**
- Resource: **Database Page**
- Select Database: your inventory reorder database
- Properties:
  - Map **Title/Name** to a string (recommended), e.g.:
    - `={{ 'Inventory run ' + $now.toISO() }}`
  - Optionally store JSON in a rich text property instead of the title.
- Connect: Merge Code → Notion node

14) **Add Asana task creation**
- Add node: **Asana**
- Name: *Create Asana Task for Inventory Db*
- Credentials: **Asana OAuth2**
- Workspace + Project: select your ops project
- Task name: build from `results[0]` or (recommended) split items and create one task per critical item.
- Due date: `={{ $now.plus({ days: 1 }).toISODate() }}`
- Connect: Merge Code → Asana node

15) **Add error handling**
- Add node: **Error Trigger**
- Name: *Workflow Error Handler*
- Add node: **Gmail**
- Name: *Send a message1*
- Subject: “Workflow Error Alert”
- Message (text) containing:
  - `{{ $json.node.name }}`
  - `{{ $json.error.message }}`
  - `{{ $now.toISO() }}`
- Connect: Error Trigger → Gmail error alert node

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Tools used: Airtable, GPT-4o (Azure), Google Sheets, Slack, Gmail, Notion, Asana. Use n8n credentials. Never hard-code API keys.” | From sticky note “🔐 Required Credentials” |
| Workflow design intent: strict JSON, no hallucinations, formulas enforced; invalid rows logged for audit readiness; multi-channel reporting | From main sticky note “📦 Automate Inventory Reordering…” |
| Slack node note claims channel `#product-faqs`, but configuration targets a **user** | Verify Slack destination setting |
| Gmail “SendTo” appears empty and subject is unrelated (“Consent Manager Governance”) | Must be corrected for production use |
| Potential data-shape risk: merge Code node stringifies output and JSON.parse; may drop structured outputs when `hasOutputParser: true` | Consider adjusting merge logic to handle object outputs safely |