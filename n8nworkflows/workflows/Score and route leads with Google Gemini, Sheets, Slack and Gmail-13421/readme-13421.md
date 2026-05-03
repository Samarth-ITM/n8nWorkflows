Score and route leads with Google Gemini, Sheets, Slack and Gmail

https://n8nworkflows.xyz/workflows/score-and-route-leads-with-google-gemini--sheets--slack-and-gmail-13421


# Score and route leads with Google Gemini, Sheets, Slack and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Score and route leads with Google Gemini, Sheets, Slack and Gmail

**Purpose:**  
Capture inbound sales leads via an n8n Form, normalize/enrich the data, score and qualify the lead with **Google Gemini**, route “hot” vs “non-hot” leads, predict revenue potential, then **log results to Google Sheets**, **alert Sales in Slack** (hot leads), and **email a lead report via Gmail**.

**Primary use cases**
- Automated lead qualification and prioritization for sales teams
- Lightweight “Revenue Intelligence” dashboarding in Google Sheets
- Real-time notification for high-intent prospects

### 1.1 Input Reception & Normalization
Form submission → normalize fields, compute budget tier, generate lead ID and timestamps.

### 1.2 Enrichment
Call an external enrichment API using the extracted email domain.

### 1.3 AI Scoring (Gemini)
Send normalized/enriched data to Gemini via a LangChain “Chain” node; parse the model output into structured lead-scoring fields.

### 1.4 Routing (Hot vs Not)
If leadScore ≥ 70 → treat as hot lead; otherwise route as normal lead.

### 1.5 Prediction, Logging & Notifications
Hot leads: Slack alert + update dashboard + revenue prediction + email report.  
Non-hot leads: update dashboard only.

---

## 2. Block-by-Block Analysis

### Block 1 — Documentation / Visual Map
**Overview:** Sticky notes provide an at-a-glance description of steps and required credentials. No runtime impact.  
**Nodes involved:**  
- Workflow Overview (Sticky Note)  
- Step 1 - Lead Capture (Sticky Note)  
- Step 2 - AI Analysis (Sticky Note)  
- Step 3 - Prediction & Routing (Sticky Note)  
- Step 4 - CRM & Notifications (Sticky Note)

#### Node: Workflow Overview
- **Type / role:** Sticky Note; describes end-to-end business logic and setup prerequisites.
- **Config choices:** Large note with sections “How it works”, “Setup steps”, “Key features”.
- **Connections:** None (non-executable).
- **Edge cases:** None.

#### Node: Step 1 - Lead Capture
- **Type / role:** Sticky Note; documents capture/enrichment step.
- **Connections:** None.

#### Node: Step 2 - AI Analysis
- **Type / role:** Sticky Note; documents Gemini scoring.
- **Connections:** None.

#### Node: Step 3 - Prediction & Routing
- **Type / role:** Sticky Note; documents parsing/routing/revenue prediction.
- **Connections:** None.

#### Node: Step 4 - CRM & Notifications
- **Type / role:** Sticky Note; documents Sheets logging + Slack + Gmail.
- **Connections:** None.

---

### Block 2 — Lead Capture & Normalization
**Overview:** Captures lead fields from a hosted n8n Form and normalizes them into consistent JSON fields (slug, budget parsing, tiers, leadId).  
**Nodes involved:**  
- Lead Capture Form  
- Process Lead Data

#### Node: Lead Capture Form
- **Type / role:** `formTrigger`; entry point collecting lead details.
- **Configuration choices (interpreted):**
  - Hosted form path: `c6eaeba6-e2e3-4bee-a954-a80948c3df15`
  - Required fields:
    - Company Name (string)
    - Contact Email (string)
    - Industry (dropdown: Technology, Healthcare, Finance, Retail, Manufacturing, Education, Other)
    - Estimated Budget (string)
    - Needs Description (string)
- **Outputs:** Sends submitted fields as JSON to **Process Lead Data**.
- **Version requirements:** TypeVersion `2.1` (Form Trigger v2+ behavior).
- **Edge cases / failures:**
  - Invalid email format not enforced unless you add validation (currently “required” only).
  - “Estimated Budget” is string; may contain currency symbols/commas—handled downstream.

#### Node: Process Lead Data
- **Type / role:** `code`; transforms raw form fields into normalized lead record.
- **Key transformations:**
  - `companyName`: trimmed
  - `contactEmail`: trimmed + lowercased
  - `industry`: defaults to `"Other"`
  - `budget`: numeric parsed from string (strips non-numeric except dot)
  - `budgetFormatted`: `$` + `toLocaleString` with 2 decimals
  - `budgetTier`:
    - `Enterprise` if ≥ 100000
    - `Mid-Market` if ≥ 25000
    - else `SMB`
  - `companySlug`: lowercased slugified company name
  - `emailDomain`: part after `@`
  - `capturedAt`: ISO timestamp
  - `leadId`: `LEAD-${Date.now()}-${RANDOM6}`
- **Inputs:** Form item(s).
- **Outputs:** Normalized JSON → **Enrich Lead Data**.
- **Version requirements:** TypeVersion `2` (Code node).
- **Edge cases / failures:**
  - Empty company name yields empty slug.
  - Missing/invalid email produces empty `emailDomain`, causing enrichment URL issues.
  - Budget parsing: `12,000 USD` becomes `12000`; `12.000,50` (EU format) may parse incorrectly.

---

### Block 3 — Enrichment
**Overview:** Enriches lead data by querying an external company enrichment API using the email domain.  
**Nodes involved:**  
- Enrich Lead Data

#### Node: Enrich Lead Data
- **Type / role:** `httpRequest`; calls enrichment API.
- **Configuration choices (interpreted):**
  - GET request to:  
    `https://api.company-enrichment.example.com/v1/lookup?domain={{ $json.emailDomain }}`
  - Timeout: 10,000 ms
- **Inputs:** Normalized lead JSON from **Process Lead Data**.
- **Outputs:** Response JSON → **AI Lead Analyzer**.
- **Version requirements:** TypeVersion `4.2` (HTTP Request node).
- **Edge cases / failures:**
  - DNS/timeout/5xx errors from the enrichment provider.
  - Empty `emailDomain` can yield a bad request or meaningless enrichment.
  - Authentication is not configured in the provided node (likely required in real use).
  - If the API returns non-JSON or unexpected schema, downstream prompting/parsing may degrade.

---

### Block 4 — AI Analysis & Scoring (Gemini via LangChain)
**Overview:** Uses Gemini to analyze lead intent and produce a scoring payload, then parses it into structured fields and grades.  
**Nodes involved:**  
- AI Lead Analyzer (Chain)  
- Google Gemini Chat Model  
- Parse AI Scoring

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini`; provides the language model for the chain.
- **Configuration choices:**
  - Model: `models/gemini-1.5-flash`
  - Temperature: `0.3` (more deterministic)
- **Connections:**
  - Output (ai_languageModel) → **AI Lead Analyzer** (ai_languageModel input)
- **Credentials required:** Google Gemini / Google AI Studio API key (configured in n8n credentials).
- **Edge cases / failures:**
  - Invalid API key / project restrictions.
  - Model name availability differences by region/account.
  - Rate limits / quota exceeded.

#### Node: AI Lead Analyzer
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm`; runs an LLM chain to generate scoring text.
- **Configuration choices:** The JSON shows empty `parameters`—meaning the chain prompt/inputs are not visible here. In practice, this node must be configured with:
  - A prompt instructing Gemini to return **machine-parseable JSON** (important because the next step extracts `{...}`).
  - Inclusion of lead/enrichment fields in the prompt context.
- **Inputs:** Enrichment response (and/or merged lead fields depending on chain configuration).
- **Outputs:** Textual model output → **Parse AI Scoring**.
- **Version requirements:** TypeVersion `1.4` (LangChain node versions can be sensitive to n8n release).
- **Edge cases / failures:**
  - If prompt not set to output JSON, parsing may fail downstream.
  - If the chain is not mapped to include lead fields, model output may be generic.
  - Large enrichment payloads can increase latency/cost or hit token limits.

#### Node: Parse AI Scoring
- **Type / role:** `code`; extracts JSON from the model response, normalizes fields, computes hot lead + grade.
- **Key logic:**
  - Reads AI output from `item.json.text` or `item.json.response` else stringifies.
  - Regex extracts first `{ ... }` block and `JSON.parse()` it.
  - On parse failure: uses fallback scoring object (leadScore 50, “manual review required” messaging).
  - Computes:
    - `leadScore` (default 50)
    - `isHotLead`: `leadScore >= 70`
    - `leadGrade`: A/B/C/D by thresholds 80/60/40
  - **Important dependency:** `const leadData = $('Process Lead Data').first().json;`  
    This pulls the first item from **Process Lead Data** node, not necessarily aligned per-item if batching/multiple items occur.
- **Inputs:** AI Lead Analyzer output.
- **Outputs:** Combined lead record + scoring → **Lead Quality Router**.
- **Edge cases / failures:**
  - If AI returns JSON with trailing comments or code fences, regex/parse may fail.
  - If AI outputs multiple JSON objects, regex may capture too much/too little.
  - The cross-node reference to `Process Lead Data` can mismatch items if multiple leads are processed concurrently or in batches (safer approach: carry lead data through the chain rather than re-reading another node).

---

### Block 5 — Routing (Hot vs Not)
**Overview:** Routes leads based on numeric score threshold. Hot leads go to Slack + revenue prediction + Sheets; non-hot go to Sheets only.  
**Nodes involved:**  
- Lead Quality Router (IF)

#### Node: Lead Quality Router
- **Type / role:** `if`; checks whether `leadScore >= 70`.
- **Configuration choices:**
  - Condition: `={{ $json.leadScore }}` **gte** `70`
  - Strict type validation enabled (as per node options)
- **Inputs:** Parsed scoring JSON from **Parse AI Scoring**.
- **Outputs:**
  - **True branch (hot):** Revenue Predictor, Sales Team Alert, Update Revenue Dashboard
  - **False branch:** Update Revenue Dashboard
- **Version requirements:** TypeVersion `2`.
- **Edge cases / failures:**
  - If `leadScore` is missing or non-numeric string, strict validation can cause unexpected behavior; current parser ensures an integer fallback (50), reducing risk.

---

### Block 6 — Prediction, Logging & Notifications
**Overview:** For hot leads, predicts revenue/sales cycle via Gemini, alerts Slack, logs to Google Sheets, and emails an HTML report via Gmail. For non-hot leads, logs to Sheets only.  
**Nodes involved:**  
- Revenue Predictor (Chain)  
- Google Gemini Chat Model1  
- Sales Team Alert (Slack)  
- Update Revenue Dashboard (Google Sheets)  
- Send Lead Report (Gmail)

#### Node: Google Gemini Chat Model1
- **Type / role:** `lmChatGoogleGemini`; language model for revenue prediction chain.
- **Configuration choices:**
  - Model: `models/gemini-1.5-flash`
  - Temperature: `0.4` (slightly more creative)
- **Connections:**
  - Output (ai_languageModel) → **Revenue Predictor**
- **Credentials required:** Same Gemini credentials as above.
- **Edge cases:** Same as above (quota, key, model availability).

#### Node: Revenue Predictor
- **Type / role:** `chainLlm`; uses Gemini to predict revenue potential / sales cycle (as described by sticky notes).
- **Configuration choices:** Parameters are empty in JSON, so actual prompt/mapping must be configured manually.
- **Inputs:** Hot-lead JSON from **Lead Quality Router** true branch.
- **Outputs:** → **Send Lead Report**
- **Version requirements:** TypeVersion `1.4`.
- **Edge cases / failures:**
  - If the chain output isn’t merged back into lead JSON, the email may lack predicted fields (currently the email template only references existing lead/scoring fields).
  - Prompt should enforce structured output if you intend to store prediction fields in Sheets.

#### Node: Sales Team Alert
- **Type / role:** `slack`; posts a formatted alert message for hot leads.
- **Configuration choices (interpreted):**
  - Sends message with lead details:
    - leadId, company, industry, budgetFormatted, leadScore/grade, purchaseIntent, dealProbability, aiSummary, recommendedActions
  - Recommended actions rendered via: `{{ $json.recommendedActions.join('\n• ') }}`
- **Inputs:** Hot-lead JSON from IF true branch.
- **Outputs:** None used downstream.
- **Credentials required:** Slack authentication (bot token) or webhook depending on node setup (n8n Slack node typically uses OAuth2; the presence of `webhookId` here is internal to n8n, not a Slack webhook URL).
- **Edge cases / failures:**
  - `recommendedActions` empty array → join yields empty string (message still sends).
  - Slack rate limiting; channel permissions; invalid auth scopes.

#### Node: Update Revenue Dashboard
- **Type / role:** `googleSheets`; logs lead record to a Google Sheet dashboard.
- **Configuration choices:**
  - Document ID: not set (empty)
  - Sheet name: not set (empty)
  - Operation is not visible; must be configured (typically “Append” row).
- **Inputs:**
  - Hot leads: from IF true branch
  - Non-hot leads: from IF false branch
- **Outputs:** None used downstream.
- **Credentials required:** Google Sheets OAuth2 / Service Account credentials in n8n.
- **Edge cases / failures:**
  - Missing documentId/sheetName will prevent execution.
  - Column mismatch if appending with “auto-map” and sheet headers don’t match JSON keys.
  - Google API quotas; permission issues on the spreadsheet.

#### Node: Send Lead Report
- **Type / role:** `gmail`; sends an HTML email report.
- **Configuration choices (interpreted):**
  - **Subject:** `Lead Report: {{ companyName }} (Score: {{ leadScore }})`
  - **Message (HTML):** includes lead fields + AI summary + recommended actions as `<li>` list.
  - Recipient (“to”) is not shown in JSON; must be configured in node UI (required for sending).
- **Inputs:** From **Revenue Predictor** (hot leads only).
- **Credentials required:** Gmail OAuth2 credentials in n8n.
- **Edge cases / failures:**
  - If “To” is not configured, node fails.
  - Gmail send limits, auth revocation, domain policy restrictions.
  - If `recommendedActions` is not an array, `.map()` may fail (parser sets it to `[]` by default, mitigating risk).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation of workflow purpose/setup |  |  | ## 💰 AI Revenue Intelligence: 360° Customer Journey Automation … ⚠️ Note: requires valid API credentials for Google Gemini, Google Sheets, Slack, and Gmail. |
| Step 1 - Lead Capture | Sticky Note | Documents capture/enrichment step |  |  | ### 📥 Step 1 - Lead Capture & Enrichment Lead submits details via web form. Data is processed and enriched with external API data. |
| Step 2 - AI Analysis | Sticky Note | Documents AI analysis/scoring |  |  | ### 🤖 Step 2 - AI Analysis & Scoring Google Gemini AI analyzes purchase intent, scores the lead, and recommends next actions. |
| Step 3 - Prediction & Routing | Sticky Note | Documents routing and prediction |  |  | ### 📊 Step 3 - Revenue Prediction & Routing AI scoring is parsed, leads are routed by quality, and revenue is predicted. |
| Step 4 - CRM & Notifications | Sticky Note | Documents Sheets/Slack/Gmail actions |  |  | ### 📧 Step 4 - CRM Update & Notifications Leads are logged to Google Sheets dashboard, hot leads trigger Slack alerts, and reports are emailed. |
| Lead Capture Form | Form Trigger | Lead intake via web form |  | Process Lead Data | ### 📥 Step 1 - Lead Capture & Enrichment Lead submits details via web form. Data is processed and enriched with external API data. |
| Process Lead Data | Code | Normalize fields, parse budget, generate leadId | Lead Capture Form | Enrich Lead Data | ### 📥 Step 1 - Lead Capture & Enrichment Lead submits details via web form. Data is processed and enriched with external API data. |
| Enrich Lead Data | HTTP Request | Company enrichment lookup by email domain | Process Lead Data | AI Lead Analyzer | ### 📥 Step 1 - Lead Capture & Enrichment Lead submits details via web form. Data is processed and enriched with external API data. |
| AI Lead Analyzer | LangChain Chain (LLM) | Gemini-based lead scoring generation | Enrich Lead Data + Gemini Model | Parse AI Scoring | ### 🤖 Step 2 - AI Analysis & Scoring Google Gemini AI analyzes purchase intent, scores the lead, and recommends next actions. |
| Google Gemini Chat Model | Google Gemini Chat Model | LLM provider for lead scoring chain |  | AI Lead Analyzer | ### 🤖 Step 2 - AI Analysis & Scoring Google Gemini AI analyzes purchase intent, scores the lead, and recommends next actions. |
| Parse AI Scoring | Code | Parse/normalize AI scoring output; compute grade/hot flag | AI Lead Analyzer | Lead Quality Router | ### 📊 Step 3 - Revenue Prediction & Routing AI scoring is parsed, leads are routed by quality, and revenue is predicted. |
| Lead Quality Router | IF | Route hot vs non-hot leads (score ≥ 70) | Parse AI Scoring | (true) Revenue Predictor; Sales Team Alert; Update Revenue Dashboard / (false) Update Revenue Dashboard | ### 📊 Step 3 - Revenue Prediction & Routing AI scoring is parsed, leads are routed by quality, and revenue is predicted. |
| Revenue Predictor | LangChain Chain (LLM) | Predict revenue/sales cycle for hot leads | Lead Quality Router (true) + Gemini Model1 | Send Lead Report | ### 📊 Step 3 - Revenue Prediction & Routing AI scoring is parsed, leads are routed by quality, and revenue is predicted. |
| Google Gemini Chat Model1 | Google Gemini Chat Model | LLM provider for revenue prediction chain |  | Revenue Predictor | ### 📊 Step 3 - Revenue Prediction & Routing AI scoring is parsed, leads are routed by quality, and revenue is predicted. |
| Sales Team Alert | Slack | Post hot lead alert to sales channel | Lead Quality Router (true) |  | ### 📧 Step 4 - CRM Update & Notifications Leads are logged to Google Sheets dashboard, hot leads trigger Slack alerts, and reports are emailed. |
| Update Revenue Dashboard | Google Sheets | Append/update dashboard rows | Lead Quality Router (true + false) |  | ### 📧 Step 4 - CRM Update & Notifications Leads are logged to Google Sheets dashboard, hot leads trigger Slack alerts, and reports are emailed. |
| Send Lead Report | Gmail | Email HTML lead report (hot leads) | Revenue Predictor |  | ### 📧 Step 4 - CRM Update & Notifications Leads are logged to Google Sheets dashboard, hot leads trigger Slack alerts, and reports are emailed. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add Form Trigger**
   - Node: **Form Trigger** (name: “Lead Capture Form”)
   - Add fields (all required):
     - Company Name (string)
     - Contact Email (string)
     - Industry (dropdown with: Technology, Healthcare, Finance, Retail, Manufacturing, Education, Other)
     - Estimated Budget (string)
     - Needs Description (string)
   - Publish/enable the form as needed (n8n provides a form URL).

3) **Add Code node: normalization**
   - Node: **Code** (name: “Process Lead Data”)
   - Paste logic that:
     - trims company name
     - lowercases email
     - extracts emailDomain
     - parses budget to float
     - builds budgetFormatted, budgetTier
     - generates companySlug
     - sets capturedAt + leadId
   - Connect: **Lead Capture Form → Process Lead Data**

4) **Add HTTP Request node: enrichment**
   - Node: **HTTP Request** (name: “Enrich Lead Data”)
   - Method: GET
   - URL: `https://api.company-enrichment.example.com/v1/lookup?domain={{ $json.emailDomain }}`
   - Timeout: 10000ms
   - Add authentication if your enrichment provider requires it (API key header, OAuth2, etc.).
   - Connect: **Process Lead Data → Enrich Lead Data**

5) **Add Gemini model for scoring**
   - Node: **Google Gemini Chat Model** (name: “Google Gemini Chat Model”)
   - Model: `models/gemini-1.5-flash`
   - Temperature: `0.3`
   - Configure **Google Gemini credentials** in n8n (API key).

6) **Add LangChain “Chain LLM” for lead scoring**
   - Node: **Chain LLM** (name: “AI Lead Analyzer”)
   - Connect the model:
     - **Google Gemini Chat Model (ai_languageModel) → AI Lead Analyzer (ai_languageModel)**
   - In the chain configuration, create a prompt that instructs Gemini to output **JSON only**, for example:
     - `leadScore` (0–100), `purchaseIntent`, `buyerPersona`, `painPoints` (array), `recommendedActions` (array), `competitivePosition`, `dealProbability` (0–100), `estimatedDealSize`, `summary`.
   - Map input variables from **Enrich Lead Data** (and/or include original lead fields if you pass them through/merge).
   - Connect: **Enrich Lead Data → AI Lead Analyzer**

7) **Add Code node: parse scoring**
   - Node: **Code** (name: “Parse AI Scoring”)
   - Implement parsing:
     - extract `{...}` from model output
     - JSON.parse with fallback object on failure
     - merge with lead data
     - compute `isHotLead` and `leadGrade`
   - Keep or revise the cross-node reference:
     - Current workflow uses `$('Process Lead Data').first().json`
     - For robustness, prefer carrying lead data forward and merging before/inside the chain.
   - Connect: **AI Lead Analyzer → Parse AI Scoring**

8) **Add IF node: routing**
   - Node: **IF** (name: “Lead Quality Router”)
   - Condition: `{{ $json.leadScore }}` **greater than or equal** `70`
   - Connect: **Parse AI Scoring → Lead Quality Router**

9) **Add Google Sheets node: dashboard logging**
   - Node: **Google Sheets** (name: “Update Revenue Dashboard”)
   - Credentials: Google Sheets OAuth2 / Service Account
   - Select **Spreadsheet (Document ID)** and **Sheet name**
   - Choose an operation (commonly **Append row**) and map fields you want to store (leadId, companyName, email, industry, budget, leadScore, grade, probability, timestamps, etc.).
   - Connect:
     - **Lead Quality Router (true) → Update Revenue Dashboard**
     - **Lead Quality Router (false) → Update Revenue Dashboard**

10) **Add Slack node: hot lead alert**
   - Node: **Slack** (name: “Sales Team Alert”)
   - Credentials: Slack OAuth2 (bot token) with permission to post to the target channel
   - Operation: Post message to channel (configure channel)
   - Message text: include lead fields; ensure `recommendedActions` is an array before joining.
   - Connect: **Lead Quality Router (true) → Sales Team Alert**

11) **Add Gemini model for revenue prediction**
   - Node: **Google Gemini Chat Model** (name: “Google Gemini Chat Model1”)
   - Model: `models/gemini-1.5-flash`
   - Temperature: `0.4`
   - Credentials: same Gemini credential (or another, if desired)

12) **Add LangChain “Chain LLM” for revenue prediction**
   - Node: **Chain LLM** (name: “Revenue Predictor”)
   - Connect model:
     - **Google Gemini Chat Model1 (ai_languageModel) → Revenue Predictor (ai_languageModel)**
   - Configure prompt to output prediction fields you need (e.g., expected ARR, salesCycleDays, nextBestStep).
   - Connect: **Lead Quality Router (true) → Revenue Predictor**

13) **Add Gmail node: send email report**
   - Node: **Gmail** (name: “Send Lead Report”)
   - Credentials: Gmail OAuth2
   - Set **To** (e.g., sales@yourcompany.com or owner email)
   - Subject: `Lead Report: {{ $json.companyName }} (Score: {{ $json.leadScore }})`
   - HTML message body: include lead details, `aiSummary`, and list `recommendedActions`.
   - Connect: **Revenue Predictor → Send Lead Report**

14) **(Optional) Add/position sticky notes**
   - Add sticky notes with the same content to document setup requirements and step grouping.

15) **Activate workflow**
   - Test by submitting the form.
   - Confirm: enrichment returns data, Gemini scoring returns JSON, Sheets row appended, Slack alert triggers only for score ≥ 70, Gmail sends successfully.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow requires valid API credentials for Google Gemini, Google Sheets, Slack, and Gmail. | Mentioned in “Workflow Overview” sticky note. |
| Enrichment API endpoint uses `api.company-enrichment.example.com` which appears to be a placeholder; replace with a real provider and configure auth. | Enrich Lead Data node URL. |
| AI parsing expects JSON embedded in the model output (`{ ... }`). Ensure the chain prompt enforces JSON-only output to avoid parse failures. | Parse AI Scoring node behavior. |
| Current parsing code references `$('Process Lead Data').first()` which may not be safe for multi-item execution; consider passing lead data forward instead. | Parse AI Scoring node dependency design note. |