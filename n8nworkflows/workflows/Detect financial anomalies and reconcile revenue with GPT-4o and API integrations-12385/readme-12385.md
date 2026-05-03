Detect financial anomalies and reconcile revenue with GPT-4o and API integrations

https://n8nworkflows.xyz/workflows/detect-financial-anomalies-and-reconcile-revenue-with-gpt-4o-and-api-integrations-12385


# Detect financial anomalies and reconcile revenue with GPT-4o and API integrations

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow runs monthly to fetch recent financial transactions, detect anomalies with GPT‑4o (plus supporting tools), verify suspected issues via a second GPT‑4o agent, then (a) automatically post approved revenue corrections to a revenue system, (b) notify a tax agent system, and (c) email a detailed alert report.

**Target use cases:** monthly close automation, revenue recognition validation, early detection of double charges, missing invoices, ghost payments, and statistical outliers.

### 1.1 Scheduled Input & Runtime Configuration
- Monthly trigger starts the run.
- A configuration node centralizes API endpoints, alert email, and time window (months).

### 1.2 Transaction Retrieval (Source of Truth)
- HTTP request pulls transactions for the last *N* months from the financial system API.

### 1.3 AI Anomaly Detection + Tooling
- A LangChain Agent powered by GPT‑4o analyzes transactions.
- It can call:
  - a calculator tool (math validation),
  - a code tool for statistical/historical pattern analysis,
  - a verification “agent tool” that delegates anomaly validation to a second GPT‑4o model.
- Agent output is forced into a structured schema via an output parser.

### 1.4 Decisioning, Auto-Corrections, and Notifications
- If anomalies were detected:
  - Build an “adjustments” array from anomalies that are verified and don’t require human review.
  - POST those adjustments to the revenue system.
  - POST a completion payload to a tax agent endpoint.
  - Send an HTML email summarizing anomalies and recommended actions.

> Note: The current wiring sends the email on the “anomalies detected” path (not the “no anomalies” path).

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Input & Workflow Configuration

**Overview:** Kicks off monthly at a defined hour, then sets reusable configuration values (API URLs, alert email, analysis window) that downstream nodes reference via expressions.

**Nodes involved:**
- Monthly Financial Data Collection
- Workflow Configuration

#### Node: **Monthly Financial Data Collection**
- **Type / Role:** `scheduleTrigger` — time-based workflow entrypoint.
- **Configuration choices:**
  - Runs every month, **at 02:00** (server timezone).
- **Input/Output:**
  - **Input:** none (trigger).
  - **Output:** to **Workflow Configuration**.
- **Version specifics:** typeVersion **1.3**.
- **Potential failures / edge cases:**
  - Timezone expectations (n8n instance timezone vs accounting timezone).
  - Missed runs if instance is down at trigger time (depends on n8n scheduling reliability/setup).

#### Node: **Workflow Configuration**
- **Type / Role:** `set` — central runtime configuration registry.
- **Configuration choices (interpreted):**
  - Creates/overrides fields:
    - `financialApiUrl` (placeholder)
    - `revenueSystemApiUrl` (placeholder)
    - `taxAgentApiUrl` (placeholder)
    - `alertEmail` (placeholder)
    - `analysisMonths` = **6**
  - **includeOtherFields: true** keeps any prior fields from trigger (if any).
- **Key expressions/variables used downstream:**
  - Referenced by other nodes via: `$('Workflow Configuration').first().json.<field>`
- **Input/Output:**
  - **Input:** from Schedule trigger.
  - **Output:** to **Fetch Financial Transactions**.
- **Version specifics:** typeVersion **3.4**.
- **Potential failures / edge cases:**
  - Leaving placeholder values will cause HTTP/email failures downstream.
  - If multiple items ever enter this node, `.first()` references only the first item.

**Sticky notes covering this block (applies to nodes here):**
- “Scheduled trigger collects monthly financial data automatically…”
- “How It Works…” (global description)

---

### Block 2 — Fetch Transactions from Financial System API

**Overview:** Retrieves transaction data for the configured lookback period (months) from the financial system, passing the resulting JSON dataset to the AI agent.

**Nodes involved:**
- Fetch Financial Transactions

#### Node: **Fetch Financial Transactions**
- **Type / Role:** `httpRequest` — pulls financial transactions.
- **Configuration choices (interpreted):**
  - **URL:** from config: `financialApiUrl`
  - **Query parameter:** `months` = `analysisMonths` (default 6)
  - Uses “Send Query” enabled.
  - Method defaults to **GET** (not explicitly set).
- **Key expressions:**
  - `url: {{ $('Workflow Configuration').first().json.financialApiUrl }}`
  - `months: {{ $('Workflow Configuration').first().json.analysisMonths }}`
- **Input/Output:**
  - **Input:** from **Workflow Configuration**
  - **Output:** to **Anomaly Detection Agent**
- **Version specifics:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - Authentication not configured (this node has no auth settings in JSON).
  - API schema mismatch: the AI agent expects transaction fields like `amount`, `category`, `id`/`transactionId`.
  - Pagination: if the API paginates results and this node doesn’t handle it, anomalies may be missed.
  - Timeouts / rate limits; unexpected non-JSON response.
  - If the API returns `{ data: [...] }` rather than an array, your downstream tool code may need adaptation.

**Sticky notes covering this block:**
- “Transaction fetcher retrieves complete financial records via API…”

---

### Block 3 — AI Anomaly Detection, Pattern Analysis, and Verification Delegation

**Overview:** A GPT‑4o-powered agent analyzes transactions, uses tools to compute statistics and validate math, and delegates per-anomaly verification to a second GPT‑4o agent tool. Outputs are enforced into JSON structures through output parsers.

**Nodes involved:**
- Anomaly Detection Agent
- OpenAI Model - Anomaly Detector
- Anomaly Detection Output Parser
- Calculator Tool
- Historical Pattern Analysis Tool
- Verification Agent Tool
- OpenAI Model - Verification Agent
- Verification Output Parser

#### Node: **OpenAI Model - Anomaly Detector**
- **Type / Role:** `lmChatOpenAi` — provides the language model for the main anomaly detection agent.
- **Configuration choices:**
  - Model: **gpt-4o**
  - Credentials: **OpenAi account** (OpenAI API credential)
- **Input/Output connections:**
  - Connected as `ai_languageModel` to **Anomaly Detection Agent**.
- **Version specifics:** typeVersion **1.3**.
- **Potential failures / edge cases:**
  - Missing/invalid OpenAI credentials; quota limits.
  - Model availability changes; organization policy restrictions.

#### Node: **Calculator Tool**
- **Type / Role:** `toolCalculator` — arithmetic tool accessible to the agent.
- **Configuration choices:** default calculator tool.
- **Connections:**
  - Exposed as `ai_tool` to **Anomaly Detection Agent**.
- **Version specifics:** typeVersion **1**.
- **Potential failures / edge cases:**
  - Generally low risk; issues mainly from agent prompting/tool selection.

#### Node: **Historical Pattern Analysis Tool**
- **Type / Role:** `toolCode` — custom JS tool to compute descriptive statistics, category frequencies, and outliers.
- **Configuration choices (interpreted):**
  - Expects `query` to be either a JSON string or an already-parsed object/array.
  - Calculates mean, variance, std dev; flags outliers beyond **2× standard deviation**.
  - Returns a JSON string with:
    - `totalTransactions`, `statistics`, `frequencyDistribution`, `outliers`, `anomalyCount`.
- **Important implementation details / assumptions:**
  - Treats missing amounts as `0`.
  - Uses `t.id` for outlier IDs; if your transactions use `transactionId`, results may be less useful unless normalized.
  - Uses `t.category || 'uncategorized'`.
- **Connections:**
  - Exposed as `ai_tool` to **Anomaly Detection Agent**.
- **Version specifics:** typeVersion **1.3**.
- **Potential failures / edge cases:**
  - If `query` is not defined (agent doesn’t supply it), tool may error or return “No transaction data…”.
  - If transactions contain non-numeric `amount`, `parseFloat` can yield `NaN` leading to `mean/stdDev` becoming `NaN`.
  - If all amounts are identical, `stdDev` can be `0`, making the outlier rule `> 0` potentially flag nothing (or behave oddly with NaN).

#### Node: **Verification Agent Tool**
- **Type / Role:** `agentTool` — a tool callable by the main agent to run a separate verification agent.
- **Configuration choices (interpreted):**
  - Tool prompt text uses `$fromAI('anomaly_data', ...)` meaning the main agent must pass a variable named `anomaly_data`.
  - System message instructs the verifier to:
    - confirm true issue vs false positive,
    - decide if human review is needed (complexity, impact, confidence),
    - compute correction amount,
    - return structured output.
- **Output parsing:** enabled (`hasOutputParser: true`) and linked to **Verification Output Parser**.
- **Connections:**
  - `ai_tool` → **Anomaly Detection Agent**
  - Uses `ai_languageModel` from **OpenAI Model - Verification Agent**
- **Version specifics:** typeVersion **3**.
- **Potential failures / edge cases:**
  - If the main agent fails to pass `anomaly_data`, the tool prompt may be incomplete and degrade verification quality.
  - Structured output enforcement can fail if the model returns invalid JSON for the schema.

#### Node: **OpenAI Model - Verification Agent**
- **Type / Role:** `lmChatOpenAi` — model for the verification agent tool.
- **Configuration choices:**
  - Model: **gpt-4o**
  - Same OpenAI credential as anomaly detector.
- **Connections:**
  - `ai_languageModel` → **Verification Agent Tool**
- **Version specifics:** typeVersion **1.3**.
- **Potential failures / edge cases:** same as the anomaly detector model (auth/quota/model access).

#### Node: **Verification Output Parser**
- **Type / Role:** `outputParserStructured` — forces verifier outputs into a defined JSON schema.
- **Schema (interpreted):**
  - `verified` (boolean)
  - `verificationNotes` (string)
  - `requiresHumanReview` (boolean)
  - `correctionAmount` (number)
- **Connections:**
  - `ai_outputParser` → **Verification Agent Tool**
- **Version specifics:** typeVersion **1.3**.
- **Potential failures / edge cases:**
  - If model returns non-conforming types (e.g., “yes” instead of boolean), parsing may fail.
  - Missing fields might be produced depending on parser strictness.

#### Node: **Anomaly Detection Output Parser**
- **Type / Role:** `outputParserStructured` — forces the main agent’s final response into structured output.
- **Schema (interpreted):**
  - `anomaliesDetected` (boolean)
  - `anomalies` (array of objects) each with:
    - `type` (string: e.g., double_charge/missing_invoice/ghost_payment/other)
    - `transactionId` (string)
    - `amount` (number)
    - `description` (string)
    - `confidence` (number 0–100)
    - `suggestedAction` (string)
- **Connections:**
  - `ai_outputParser` → **Anomaly Detection Agent**
- **Version specifics:** typeVersion **1.3**.
- **Potential failures / edge cases:**
  - Same structured parsing risks as above.
  - If anomalies are verified via the tool, the workflow later expects `verified`, `requiresHumanReview`, `correctionAmount` to exist on each anomaly—but that is **not included** in this parser schema (see edge case below).

#### Node: **Anomaly Detection Agent**
- **Type / Role:** `agent` — orchestrates the anomaly detection process, tool calls, and final structured output.
- **Configuration choices (interpreted):**
  - **Prompt text:** `Analyze the following financial transaction data for anomalies: <stringified $json>`
  - **System message:** prescribes:
    1) call `analyze_historical_patterns` tool,
    2) detect anomaly types,
    3) call Verification Agent Tool per anomaly,
    4) compute confidence scores,
    5) suggest actions,
    6) return structured format.
  - `promptType: define`
  - Output parser enabled (`hasOutputParser: true`) → **Anomaly Detection Output Parser**
- **Tools / models connected:**
  - Language model: **OpenAI Model - Anomaly Detector**
  - Tools: **Calculator Tool**, **Historical Pattern Analysis Tool**, **Verification Agent Tool**
- **Input/Output:**
  - **Input:** transactions JSON from **Fetch Financial Transactions**
  - **Output:** to **Check for Anomalies**
- **Version specifics:** typeVersion **3.1**.
- **Potential failures / edge cases:**
  - **Schema mismatch for downstream automation:** later nodes filter anomalies using `a.verified` and `a.requiresHumanReview` and map `a.correctionAmount`, but the anomaly schema produced by the main output parser does not include these fields. Unless the agent injects extra fields beyond schema (may be dropped/blocked), the “Prepare Revenue Adjustments” node may produce an empty adjustments array or throw errors depending on runtime behavior.
  - Large transaction payloads can exceed model context limits; consider summarizing or batching.
  - Tool calling reliability: model may omit using tools if not enforced strongly; outputs may be less accurate.

**Sticky notes covering this block:**
- “Anomaly detection agent analyzes using calculator, pattern analysis, and AI models…”
- “Setup Steps…” (OpenAI credentials, thresholds, integrations)
- “Prerequisites / Use Cases / Customization / Benefits…”

---

### Block 4 — Decisioning, Revenue Updates, Tax Notification, and Email Alerting

**Overview:** If anomalies exist, the workflow prepares automatic revenue corrections for anomalies that are verified and safe to auto-adjust, posts them to the revenue system, notifies the tax agent endpoint, and emails a detailed HTML report.

**Nodes involved:**
- Check for Anomalies
- Prepare Revenue Adjustments
- Update Revenue Entries
- Notify Tax Agent
- Send Anomaly Alert

#### Node: **Check for Anomalies**
- **Type / Role:** `if` — gates downstream actions.
- **Condition (interpreted):**
  - Proceed “true” branch if `{{$json.anomaliesDetected}} === true`
- **Connections:**
  - True output goes to:
    - **Prepare Revenue Adjustments**
    - **Send Anomaly Alert**
  - No explicit false branch connected.
- **Version specifics:** typeVersion **2.3**.
- **Potential failures / edge cases:**
  - If `anomaliesDetected` is missing/null due to parsing failure, condition evaluates false and nothing else runs.
  - Loose type validation is enabled; still, malformed outputs may behave unexpectedly.

#### Node: **Prepare Revenue Adjustments**
- **Type / Role:** `set` — builds an array of adjustments for API posting.
- **Configuration choices (interpreted):**
  - Creates `adjustments` as:
    - Filter anomalies where `verified && !requiresHumanReview`
    - Map to `{ transactionId, correctionAmount, type }`
  - includeOtherFields: true (keeps anomalies for later nodes)
- **Key expression:**
  - `{{ $json.anomalies.filter(a => a.verified && !a.requiresHumanReview).map(a => ({ transactionId: a.transactionId, correctionAmount: a.correctionAmount, type: a.type })) }}`
- **Connections:**
  - Output → **Update Revenue Entries**
- **Version specifics:** typeVersion **3.4**.
- **Potential failures / edge cases:**
  - If `anomalies` is undefined/not an array: expression throws.
  - If anomalies lack `verified/requiresHumanReview/correctionAmount`: filtering results in empty list or undefined values.
  - Consider guarding with defaults (e.g., `($json.anomalies ?? [])`).

#### Node: **Update Revenue Entries**
- **Type / Role:** `httpRequest` — posts corrections to revenue system.
- **Configuration choices (interpreted):**
  - URL from config: `revenueSystemApiUrl`
  - Method: **POST**
  - JSON body:
    - `adjustments: $json.adjustments`
    - `timestamp: new Date().toISOString()`
    - `source: 'anomaly_detection_system'`
- **Connections:**
  - Output → **Notify Tax Agent**
- **Version specifics:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - Auth not configured; endpoint may reject.
  - If `adjustments` is empty, your revenue API may treat it as no-op or error.
  - Idempotency: reruns could double-apply corrections unless the API de-duplicates.

#### Node: **Notify Tax Agent**
- **Type / Role:** `httpRequest` — informs tax agent system.
- **Configuration choices (interpreted):**
  - URL from config: `taxAgentApiUrl`
  - Method: **POST**
  - JSON body includes:
    - `adjustments`
    - `anomaliesCount: $json.anomalies.length`
    - timestamp and `status: 'completed'`
- **Connections:**
  - No downstream nodes.
- **Version specifics:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - If `anomalies` missing, `.length` expression fails.
  - Tax endpoint may require additional fields (client ID, period, etc.).

#### Node: **Send Anomaly Alert**
- **Type / Role:** `emailSend` — sends HTML report to stakeholders.
- **Configuration choices (interpreted):**
  - To: `alertEmail` from config
  - Subject: “Financial Anomalies Detected - Review Required”
  - HTML body: includes summary and per-anomaly details rendered via JS `.map().join('')`
  - From: placeholder sender email
- **Connections:**
  - Receives data directly from **Check for Anomalies** true path.
- **Version specifics:** typeVersion **2.1**.
- **Potential failures / edge cases:**
  - Email credentials not shown in JSON; node may require SMTP credentials in n8n.
  - If anomalies array is very large, email size may be excessive.
  - The subject implies anomalies exist; but the node is only executed when anomaliesDetected is true, consistent with current logic.

**Sticky notes covering this block:**
- “Setup Steps…” (mentions email setup and tax system integration)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Financial Data Collection | scheduleTrigger | Monthly workflow entrypoint | — | Workflow Configuration | Scheduled trigger collects monthly financial data automatically. Why: Ensures consistent monitoring without manual intervention, catching issues promptly before month-end close |
| Workflow Configuration | set | Centralized config (URLs, email, months) | Monthly Financial Data Collection | Fetch Financial Transactions | ## Setup Steps… (OpenAI creds, API auth, thresholds, tax integration, email setup) / ## Prerequisites… / ## How It Works… / Scheduled trigger note (if visually overlapping) |
| Fetch Financial Transactions | httpRequest | Pull transactions from financial API | Workflow Configuration | Anomaly Detection Agent | Transaction fetcher retrieves complete financial records via API. Why: Accesses real-time data from source systems, eliminating data entry errors and delays |
| OpenAI Model - Anomaly Detector | lmChatOpenAi | LLM backend for anomaly agent | — (wired as AI resource) | Anomaly Detection Agent | Anomaly detection agent analyzes using calculator, pattern analysis, and AI models. Why: Combines mathematical validation with machine learning to identify diverse anomaly types human review might miss |
| Calculator Tool | toolCalculator | Math validation tool for agent | — (wired as tool) | Anomaly Detection Agent | Anomaly detection agent analyzes using calculator, pattern analysis, and AI models… |
| Historical Pattern Analysis Tool | toolCode | Statistical analysis & outlier detection tool | — (wired as tool) | Anomaly Detection Agent | Anomaly detection agent analyzes using calculator, pattern analysis, and AI models… |
| OpenAI Model - Verification Agent | lmChatOpenAi | LLM backend for verification tool | — (wired as AI resource) | Verification Agent Tool | Anomaly detection agent analyzes using calculator, pattern analysis, and AI models… |
| Verification Output Parser | outputParserStructured | Enforce verifier structured output | — (wired as parser) | Verification Agent Tool | Anomaly detection agent analyzes using calculator, pattern analysis, and AI models… |
| Verification Agent Tool | agentTool | Delegated anomaly verification callable by main agent | — (called by agent) | Anomaly Detection Agent (as tool) | Anomaly detection agent analyzes using calculator, pattern analysis, and AI models… |
| Anomaly Detection Output Parser | outputParserStructured | Enforce anomaly report schema | — (wired as parser) | Anomaly Detection Agent | Anomaly detection agent analyzes using calculator, pattern analysis, and AI models… |
| Anomaly Detection Agent | agent | Detect anomalies, call tools, compile results | Fetch Financial Transactions | Check for Anomalies | Anomaly detection agent analyzes using calculator, pattern analysis, and AI models… |
| Check for Anomalies | if | Route only when anomaliesDetected is true | Anomaly Detection Agent | Prepare Revenue Adjustments; Send Anomaly Alert | (No dedicated sticky note shown for this exact node) |
| Prepare Revenue Adjustments | set | Build auto-correction payload | Check for Anomalies (true) | Update Revenue Entries | ## Setup Steps… (mentions tax integration and email setup) |
| Update Revenue Entries | httpRequest | Post adjustments to revenue system | Prepare Revenue Adjustments | Notify Tax Agent | ## Setup Steps… (Configure tax system integration credentials in “Update Revenue Entries”) |
| Notify Tax Agent | httpRequest | Inform tax agent system of results | Update Revenue Entries | — | ## Setup Steps… |
| Send Anomaly Alert | emailSend | Email anomaly report | Check for Anomalies (true) | — | ## Setup Steps… (Set up email notification service…) |

> Sticky notes are positioned over broad regions; where overlap is ambiguous, content was applied to the most clearly related nodes.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a Schedule Trigger**
   - Node: **Schedule Trigger**
   - Name: `Monthly Financial Data Collection`
   - Set interval to **Monthly** and set **Trigger at hour = 2**.

2) **Add a Set node for configuration**
   - Node: **Set**
   - Name: `Workflow Configuration`
   - Add fields:
     - `financialApiUrl` (string) – your financial transactions endpoint
     - `revenueSystemApiUrl` (string) – revenue system corrections endpoint
     - `taxAgentApiUrl` (string) – tax agent notification endpoint
     - `alertEmail` (string) – recipient email for alerts
     - `analysisMonths` (number) – set to **6** (or desired lookback)
   - Enable **Include Other Fields**.
   - Connect: **Monthly Financial Data Collection → Workflow Configuration**

3) **Add HTTP Request to fetch transactions**
   - Node: **HTTP Request**
   - Name: `Fetch Financial Transactions`
   - Method: **GET**
   - URL expression: `{{ $('Workflow Configuration').first().json.financialApiUrl }}`
   - Enable **Send Query Parameters**
   - Add query parameter:
     - `months` = `{{ $('Workflow Configuration').first().json.analysisMonths }}`
   - Configure authentication as required by your financial API (e.g., Header Auth/Bearer, OAuth2).
   - Connect: **Workflow Configuration → Fetch Financial Transactions**

4) **Add OpenAI Chat Model for anomaly detection**
   - Node: **OpenAI Chat Model** (LangChain `lmChatOpenAi`)
   - Name: `OpenAI Model - Anomaly Detector`
   - Model: **gpt-4o**
   - Credentials: configure **OpenAI API** credential in n8n.

5) **Add tools for the anomaly agent**
   - Node: **Calculator Tool** (LangChain `toolCalculator`)
     - Name: `Calculator Tool`
   - Node: **Code Tool** (LangChain `toolCode`)
     - Name: `Historical Pattern Analysis Tool`
     - Paste the provided JS code (statistics/outliers) into the code field.
     - Description: “Analyzes historical transaction patterns…”

6) **Add OpenAI Chat Model for verification**
   - Node: **OpenAI Chat Model** (`lmChatOpenAi`)
   - Name: `OpenAI Model - Verification Agent`
   - Model: **gpt-4o**
   - Credentials: same OpenAI credential (or a separate one).

7) **Add a Structured Output Parser for verification**
   - Node: **Structured Output Parser** (`outputParserStructured`)
   - Name: `Verification Output Parser`
   - Schema (manual):
     - `verified` boolean
     - `verificationNotes` string
     - `requiresHumanReview` boolean
     - `correctionAmount` number

8) **Add Verification Agent Tool**
   - Node: **Agent Tool** (`agentTool`)
   - Name: `Verification Agent Tool`
   - Tool description: “Verifies detected financial anomalies…”
   - Prompt text expression:
     - `Verify this anomaly: {{ $fromAI('anomaly_data', 'The anomaly data to verify including type, amount, and description', 'string') }}`
   - System message: set to the verifier instructions (validate issue, human review decision, correction amount, notes).
   - Enable **Output Parser** and connect:
     - **Verification Output Parser** → as the tool’s output parser
   - Connect model resource:
     - **OpenAI Model - Verification Agent** → as the tool’s language model

9) **Add a Structured Output Parser for anomaly detection**
   - Node: **Structured Output Parser** (`outputParserStructured`)
   - Name: `Anomaly Detection Output Parser`
   - Manual schema:
     - `anomaliesDetected` boolean
     - `anomalies[]` objects with: `type`, `transactionId`, `amount`, `description`, `confidence`, `suggestedAction`

10) **Add the main Anomaly Detection Agent**
   - Node: **Agent** (LangChain `agent`)
   - Name: `Anomaly Detection Agent`
   - Prompt text expression:
     - `Analyze the following financial transaction data for anomalies: {{ JSON.stringify($json) }}`
   - System message: set to the anomaly specialist instructions (use tools, detect types, call verification tool, confidence/actions).
   - Enable **Output Parser** and connect:
     - **Anomaly Detection Output Parser** → agent output parser
   - Connect resources/tools to the agent:
     - Language model: **OpenAI Model - Anomaly Detector**
     - Tools: **Calculator Tool**, **Historical Pattern Analysis Tool**, **Verification Agent Tool**
   - Connect: **Fetch Financial Transactions → Anomaly Detection Agent**

11) **Add an IF node to check anomalies**
   - Node: **IF**
   - Name: `Check for Anomalies`
   - Condition: Boolean equals:
     - Left: `{{ $json.anomaliesDetected }}`
     - Right: `true`
   - Connect: **Anomaly Detection Agent → Check for Anomalies**

12) **Add Set node to prepare adjustments**
   - Node: **Set**
   - Name: `Prepare Revenue Adjustments`
   - Add field `adjustments` (array) with expression:
     - `{{ $json.anomalies.filter(a => a.verified && !a.requiresHumanReview).map(a => ({ transactionId: a.transactionId, correctionAmount: a.correctionAmount, type: a.type })) }}`
   - Enable **Include Other Fields**
   - Connect from **Check for Anomalies (true)** → **Prepare Revenue Adjustments**

13) **Add HTTP Request to update revenue system**
   - Node: **HTTP Request**
   - Name: `Update Revenue Entries`
   - Method: **POST**
   - URL: `{{ $('Workflow Configuration').first().json.revenueSystemApiUrl }}`
   - Body: JSON, set to:
     - `{{ { adjustments: $json.adjustments, timestamp: new Date().toISOString(), source: 'anomaly_detection_system' } }}`
   - Configure authentication as required.
   - Connect: **Prepare Revenue Adjustments → Update Revenue Entries**

14) **Add HTTP Request to notify tax agent**
   - Node: **HTTP Request**
   - Name: `Notify Tax Agent`
   - Method: **POST**
   - URL: `{{ $('Workflow Configuration').first().json.taxAgentApiUrl }}`
   - Body JSON:
     - `{{ { adjustments: $json.adjustments, anomaliesCount: $json.anomalies.length, timestamp: new Date().toISOString(), status: 'completed' } }}`
   - Configure authentication as required.
   - Connect: **Update Revenue Entries → Notify Tax Agent**

15) **Add Email Send node for alerting**
   - Node: **Email Send**
   - Name: `Send Anomaly Alert`
   - To: `{{ $('Workflow Configuration').first().json.alertEmail }}`
   - From: set a valid sender address for your SMTP/Email provider
   - Subject: “Financial Anomalies Detected - Review Required”
   - HTML: use the provided template (summary + anomaly list rendering).
   - Configure email credentials (SMTP or provider integration depending on your n8n setup).
   - Connect from **Check for Anomalies (true)** → **Send Anomaly Alert**

### Credential setup checklist
- **OpenAI**: create an OpenAI credential in n8n and select it in both OpenAI model nodes.
- **Financial API / Revenue API / Tax API**: configure auth in each HTTP Request node (Bearer token, API key header, OAuth2, etc.).
- **Email**: configure SMTP (or relevant email credential) for `emailSend`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## How It Works … collects monthly financial transactions… AI anomaly detection agent… verification by a secondary AI agent… automated revenue adjustments and tax agent notifications… alert emails…” | Sticky Note: How It Works (global workflow description) |
| “## Prerequisites OpenAI API access, financial system API credentials with read/write permissions… Use Cases… Customization… Benefits…” | Sticky Note: Prerequisites / Use Cases / Customization / Benefits |
| “## Setup Steps 1. Configure OpenAI API credentials… 2. Set up financial data source API connection… 3. Define anomaly detection thresholds… 4. Configure tax system integration… 5. Set up email notification…” | Sticky Note: Setup Steps |
| Design intent: combine mathematical validation + statistical pattern analysis + LLM reasoning | Sticky Note: Anomaly detection agent rationale |

### Important implementation caveat (recommended fix)
Downstream automation expects `verified`, `requiresHumanReview`, and `correctionAmount` on each anomaly, but the **Anomaly Detection Output Parser** schema does not define them. To make auto-adjustments work reliably, either:
- Extend the anomaly schema to include these verification fields, **or**
- Store verification results separately and merge them into anomalies before “Prepare Revenue Adjustments”.