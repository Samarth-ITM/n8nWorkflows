Monitor and alert stalled deals in Zoho CRM with Gemini AI and Gmail

https://n8nworkflows.xyz/workflows/monitor-and-alert-stalled-deals-in-zoho-crm-with-gemini-ai-and-gmail-12025


# Monitor and alert stalled deals in Zoho CRM with Gemini AI and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Monitor and alert stalled deals in Zoho CRM with Gemini AI and Gmail  
**Workflow name (in JSON):** Zoho CRM - Deal health Predictor  
**Purpose:** Runs a weekly scan of active Zoho CRM deals, calculates “stall” metrics (inactivity, stage age, deal age), uses **Google Gemini** to generate a structured health score and recommendations for stalled deals, then **emails the deal owner** and **creates a Zoho CRM task** when a deal is deemed at risk.

### 1.1 Weekly Trigger & Initial Pass-through
Schedules the workflow weekly and passes data onward (currently without adding/transforming fields in the Set node).

### 1.2 Zoho Deal Retrieval & Itemization
Fetches active deals from Zoho CRM using a search query and splits the response array into individual deal items.

### 1.3 Metric Calculation & Stall Filtering
Computes inactivity days, stage days, and deal age using timestamps from Zoho; filters deals that appear “stalled” based on inactivity.

### 1.4 AI Health Scoring (Gemini + Structured JSON)
For each stalled deal, calls Gemini with a prompt and forces structured JSON output via a schema-like parser.

### 1.5 Risk Threshold Check & Actions
If the health score is below the threshold, sends a Gmail alert to the deal owner and creates a high-priority Zoho CRM task linked to the deal.

---

## 2. Block-by-Block Analysis

### Block 1 — Weekly Trigger & Initial Pass-through

**Overview:** Starts the workflow weekly (Monday at 09:00) and routes execution to the Zoho fetch step via a Set node.

**Nodes involved:**
- Weekly Deal Health Scan Trigger
- Edit Fields

#### Node: Weekly Deal Health Scan Trigger
- **Type / role:** `Schedule Trigger` — time-based entry point.
- **Configuration (interpreted):** Runs **every week**, on **day 1 (Monday)** at **09:00** (instance timezone applies).
- **Connections:** Outputs to **Edit Fields**.
- **Edge cases / failures:**
  - Timezone mismatch (instance settings vs business expectation).
  - Workflow inactive (`active: false`) means schedule won’t run until enabled.

#### Node: Edit Fields
- **Type / role:** `Set` — field editing / shaping node (currently acts as a pass-through).
- **Configuration (interpreted):**
  - `includeOtherFields: true` and no explicit fields configured → effectively forwards input unchanged.
- **Connections:** Receives from **Weekly Deal Health Scan Trigger**, outputs to **Fetch Active Deals From Zoho**.
- **Edge cases / failures:**
  - If later modified to remove fields, downstream expressions may break (e.g., expecting data from Zoho response).

**Sticky note covering this block:**
- “## Weekly Scan Trigger  
  Runs every week to score active deals and flag those at risk.”

---

### Block 2 — Zoho Deal Retrieval & Itemization

**Overview:** Pulls active/open deals from Zoho CRM (specific stages) and converts the returned array into one item per deal.

**Nodes involved:**
- Fetch Active Deals From Zoho
- Split Deals Into Individual Items

#### Node: Fetch Active Deals From Zoho
- **Type / role:** `HTTP Request` — calls Zoho CRM API search endpoint.
- **Configuration (interpreted):**
  - **Method:** GET (default)
  - **URL:** `https://www.zohoapis.com/crm/v2/Deals/search?criteria=(Stage:in:Qualification,Negotiation,Proposal,Demo,Value Proposition)`
  - **Auth:** Predefined credential type: `zohoOAuth2Api`
  - **alwaysOutputData:** true (node returns an item even on some error conditions depending on n8n behavior)
- **Output data expectation:**
  - Zoho typically returns `{ data: [ ...deals... ], info: {...} }`. This workflow assumes `data` exists.
- **Connections:** Outputs to **Split Deals Into Individual Items**.
- **Version requirements:** TypeVersion **4.3**.
- **Edge cases / failures:**
  - Zoho OAuth token expiration / invalid refresh token.
  - Stage names must match Zoho exactly; otherwise search returns empty.
  - Pagination: Zoho search endpoints can paginate; this node as configured does **not** iterate pages—may miss deals beyond the first page.
  - Rate limits (HTTP 429) or transient 5xx errors.
  - If Zoho returns no `data`, the next node may output nothing.

#### Node: Split Deals Into Individual Items
- **Type / role:** `Split Out` — splits an array field into individual items.
- **Configuration (interpreted):**
  - Field to split: `data`
- **Connections:** Receives from **Fetch Active Deals From Zoho**, outputs to **Calculate Deal Metrics - Deal Inactivitu and Stage Age**.
- **Edge cases / failures:**
  - If `data` is missing/null/not an array, split may yield 0 items or error depending on runtime behavior.

**Sticky note covering this block:**
- “## Data Load & Stalling Filter  
  Grabs open CRM deals → enriches with inactivity metrics → only stalled deals move forward.”

---

### Block 3 — Metric Calculation & Stall Filtering

**Overview:** Adds computed metrics to each deal and filters only those considered stalled (inactivity ≥ 2 days).

**Nodes involved:**
- Calculate Deal Metrics - Deal Inactivitu and Stage Age
- Filter stalled Deals

#### Node: Calculate Deal Metrics - Deal Inactivitu and Stage Age
- **Type / role:** `Code` — computes numeric metrics from Zoho timestamps.
- **Configuration (interpreted):**
  - JavaScript calculates:
    - `inactivity_days` from `Last_Activity_Time` (days since last activity)
    - `stage_days` from `Stage_Span` or fallback `Stage_Start_Time`
    - `deal_age` from `Created_Time`
  - Adds these fields to each deal item and returns a new item array.
- **Key variables:**
  - `now = new Date()`
  - Uses `Math.floor((now - new Date(timestamp)) / (1000*60*60*24))`
- **Connections:** Receives from **Split Deals Into Individual Items**, outputs to **Filter stalled Deals**.
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - Missing timestamps → metric becomes `null`.
  - Invalid date strings → `new Date()` may yield `Invalid Date`, producing `NaN`.
  - Timezone differences in Zoho timestamps vs server time can shift day boundaries.
  - Field naming mismatch (e.g., Zoho uses different API field keys) causes all metrics to be null.

#### Node: Filter stalled Deals
- **Type / role:** `IF` — gates the flow to AI scoring only for stalled deals.
- **Configuration (interpreted):**
  - Condition: `{{ $json.inactivity_days || 0 }} >= 2`
  - Combinator: OR (but only one condition is present)
- **Connections:**
  - **True** output → **AI Deal Health Scoring**
  - False output is unused (stalled filter “drops” non-stalled deals from further processing in this design)
- **Version requirements:** TypeVersion **2.2**.
- **Edge cases / failures:**
  - If inactivity is null, expression falls back to 0; deals with unknown activity will be treated as **not stalled** (may be undesirable).
  - Business rule mismatch: threshold hard-coded to 2 days.

**Sticky note coverage:** same “Data Load & Stalling Filter” note (above).

---

### Block 4 — AI Health Scoring (Gemini + Structured JSON)

**Overview:** Uses Gemini to generate a deal health assessment as structured JSON, including risk level, reasons, recommended actions, and a suggested message.

**Nodes involved:**
- Google Gemini Chat Model
- Structured Output Parser
- AI Deal Health Scoring

#### Node: Google Gemini Chat Model
- **Type / role:** `LangChain Chat Model (Google Gemini)` — provides the LLM backend for the chain.
- **Configuration (interpreted):**
  - Uses Google PaLM/Gemini credentials (`googlePalmApi`)
  - No extra options set (defaults for model/temperature depend on node defaults and credential configuration)
- **Connections:** Connected via `ai_languageModel` to **AI Deal Health Scoring**.
- **Edge cases / failures:**
  - Credential misconfiguration (API key invalid, project restrictions).
  - Model availability/region limits.
  - Content or safety filters may block some outputs.
  - Latency/timeouts under load.

#### Node: Structured Output Parser
- **Type / role:** `LangChain Structured Output Parser` — constrains output to a JSON structure.
- **Configuration (interpreted):**
  - Provides a JSON schema example requiring:
    - `health_score` (0–100)
    - `risk_level` (Low/Medium/High)
    - `top_reasons` (array)
    - `recommended_actions` (array)
    - `suggested_message` (short 40-word message)
- **Connections:** Connected via `ai_outputParser` to **AI Deal Health Scoring**.
- **Version requirements:** TypeVersion **1.3**.
- **Edge cases / failures:**
  - Model returns non-JSON or malformed JSON → parsing fails or yields partial output.
  - `health_score` might come back as string vs number; downstream IF uses loose validation (helps, but not guaranteed).

#### Node: AI Deal Health Scoring
- **Type / role:** `LangChain LLM Chain` — orchestrates prompt + model + output parser.
- **Configuration (interpreted):**
  - Prompt injects deal fields:
    - `Stage`, `inactivity_days`, `stage_days`, `deal_age`, `Amount`, `Last_Activity_Time`, `Closing_Date`
  - Requires JSON-only output and uses the attached output parser.
  - `hasOutputParser: true`
- **Output shape (expected):**
  - The node outputs an `output` object, referenced later as `$json.output.health_score`, etc.
- **Connections:** Outputs to **Is Deal At-Risk?**.
- **Version requirements:** TypeVersion **1.7**.
- **Edge cases / failures:**
  - Missing deal fields results in “undefined” in prompt; model may behave unpredictably.
  - If Zoho fields have different names, prompt contains blanks, lowering scoring quality.
  - Token limits if Zoho fields are large (less likely here).
  - Parser mismatch (model returns numbers as strings, arrays as text, etc.).

**Sticky note covering this block:**
- “## AI Risk Evaluation  
  Gemini assigns risk score + actionable recommendations.”

---

### Block 5 — Risk Threshold Check & Actions (Gmail + Zoho Task)

**Overview:** Checks whether the AI health score indicates risk (< 60). If at risk, sends an alert email to the deal owner and creates a linked high-priority Zoho CRM task due in 3 days.

**Nodes involved:**
- Is Deal At-Risk?
- Send At-Risk Alert Email
- Create At-Risk Task (Zoho CRM)

#### Node: Is Deal At-Risk?
- **Type / role:** `IF` — threshold decision.
- **Configuration (interpreted):**
  - Condition: `{{ $json.output.health_score }} < 60`
  - Loose type validation enabled (helps if `health_score` is a string).
- **Connections:**
  - True output → **Send At-Risk Alert Email**
  - False output unused (no action for healthy deals)
- **Edge cases / failures:**
  - If `output` is missing (parser failure), expression can evaluate to null → condition may fail unexpectedly.
  - If health_score is non-numeric text, even loose conversion may not behave as intended.

#### Node: Send At-Risk Alert Email
- **Type / role:** `Gmail` — sends notification email.
- **Configuration (interpreted):**
  - **To:** `{{ $('Calculate Deal Metrics - Deal Inactivitu and Stage Age').item.json.Owner.email }}`
  - **Subject:** “Deal At-Risk Alert”
  - **Body:** Plain text containing deal name, stage, score, reasons, actions, suggested message.
  - **emailType:** text
- **Connections:** Outputs to **Create At-Risk Task (Zoho CRM)**.
- **Version requirements:** TypeVersion **2.1**.
- **Edge cases / failures:**
  - `Owner.email` missing → sendTo becomes empty → Gmail node error.
  - Gmail OAuth scopes/consent issues; token expiration.
  - Rate limits or “daily sending limit exceeded”.
  - Referencing another node’s `item` assumes item alignment; if execution becomes non-linear or merged later, this can break.

#### Node: Create At-Risk Task (Zoho CRM)
- **Type / role:** `HTTP Request` — creates a CRM task via Zoho API.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://www.zohoapis.com/crm/v2/Tasks`
  - **Auth:** Predefined `zohoOAuth2Api`
  - **Body:** JSON payload creating one task with:
    - Subject: “AI Alert : At-Risk Deal”
    - Due date: `{{ $now.plus({ days: 3 }).toISODate() }}`
    - Status: “Not Started”
    - Priority: “High”
    - `$se_module`: “Deals”
    - `What_Id`: deal id from the metrics node
    - Description includes deal details and AI score reference
  - Trigger: `workflow`
- **Connections:** Terminal node (no outputs connected).
- **Version requirements:** TypeVersion **4.3**.
- **Edge cases / failures:**
  - Invalid/missing deal id (`What_Id`) → Zoho validation error.
  - Zoho permissions: token must allow Tasks creation.
  - Date formatting mismatch (should be ISO date; `toISODate()` is appropriate).
  - Description references `$('AI Deal Health Scoring').item.json.output.health_score`; if item mapping shifts, could reference wrong item.

**Sticky note covering this block:**
- “## Alerts & CRM Actions  
  If health score < threshold → notify rep + create high-priority CRM task.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Deal Health Scan Trigger | Schedule Trigger | Weekly entry point (Monday 09:00) | — | Edit Fields | ## Weekly Scan Trigger / Runs every week to score active deals and flag those at risk. |
| Edit Fields | Set | Pass-through / field shaping placeholder | Weekly Deal Health Scan Trigger | Fetch Active Deals From Zoho | ## Data Load & Stalling Filter / Grabs open CRM deals → enriches with inactivity metrics → only stalled deals move forward. |
| Fetch Active Deals From Zoho | HTTP Request | Query Zoho CRM Deals by active stages | Edit Fields | Split Deals Into Individual Items | ## Data Load & Stalling Filter / Grabs open CRM deals → enriches with inactivity metrics → only stalled deals move forward. |
| Split Deals Into Individual Items | Split Out | Split Zoho `data[]` into per-deal items | Fetch Active Deals From Zoho | Calculate Deal Metrics - Deal Inactivitu and Stage Age | ## Data Load & Stalling Filter / Grabs open CRM deals → enriches with inactivity metrics → only stalled deals move forward. |
| Calculate Deal Metrics - Deal Inactivitu and Stage Age | Code | Compute inactivity/stage/deal age metrics | Split Deals Into Individual Items | Filter stalled Deals | ## Data Load & Stalling Filter / Grabs open CRM deals → enriches with inactivity metrics → only stalled deals move forward. |
| Filter stalled Deals | IF | Filter stalled deals (inactivity ≥ 2) | Calculate Deal Metrics - Deal Inactivitu and Stage Age | AI Deal Health Scoring (true) | ## Data Load & Stalling Filter / Grabs open CRM deals → enriches with inactivity metrics → only stalled deals move forward. |
| Google Gemini Chat Model | LangChain Chat Model (Google Gemini) | LLM provider for scoring | — | AI Deal Health Scoring (ai_languageModel) | ## AI Risk Evaluation / Gemini assigns risk score + actionable recommendations. |
| Structured Output Parser | Structured Output Parser | Enforce JSON output structure | — | AI Deal Health Scoring (ai_outputParser) | ## AI Risk Evaluation / Gemini assigns risk score + actionable recommendations. |
| AI Deal Health Scoring | LLM Chain | Prompt + model + parsed JSON scoring | Filter stalled Deals | Is Deal At-Risk? | ## AI Risk Evaluation / Gemini assigns risk score + actionable recommendations. |
| Is Deal At-Risk? | IF | Risk threshold check (score < 60) | AI Deal Health Scoring | Send At-Risk Alert Email (true) | ## Alerts & CRM Actions / If health score < threshold → notify rep + create high-priority CRM task. |
| Send At-Risk Alert Email | Gmail | Email the deal owner | Is Deal At-Risk? | Create At-Risk Task (Zoho CRM) | ## Alerts & CRM Actions / If health score < threshold → notify rep + create high-priority CRM task. |
| Create At-Risk Task (Zoho CRM) | HTTP Request | Create linked high-priority task in Zoho | Send At-Risk Alert Email | — | ## Alerts & CRM Actions / If health score < threshold → notify rep + create high-priority CRM task. |
| Sticky Note | Sticky Note | Comment (overview) | — | — | ## Zoho CRM – Deal Health Predictor (Overview) / How it works… Setup steps… |
| Sticky Note1 | Sticky Note | Comment (trigger) | — | — | ## Weekly Scan Trigger / Runs every week to score active deals and flag those at risk. |
| Sticky Note2 | Sticky Note | Comment (data + filter) | — | — | ## Data Load & Stalling Filter / Grabs open CRM deals → enriches… |
| Sticky Note3 | Sticky Note | Comment (AI) | — | — | ## AI Risk Evaluation / Gemini assigns risk score + actionable recommendations. |
| Sticky Note4 | Sticky Note | Comment (actions) | — | — | ## Alerts & CRM Actions / If health score < threshold… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“Zoho CRM - Deal health Predictor”** (or your preferred name).
- Keep it **inactive** until testing is complete.

2) **Add Schedule Trigger**
- Node: **Schedule Trigger**
- Configure: weekly → **Monday** at **09:00** (adjust timezone in n8n settings if needed).
- Connect it to the next node.

3) **Add Set node (“Edit Fields”)**
- Node: **Set**
- Enable **Include Other Fields**.
- Do not add/remove fields (acts as a pass-through).
- Connect: **Schedule Trigger → Set → HTTP Request (Zoho fetch)**

4) **Add Zoho deal fetch**
- Node: **HTTP Request**
- Authentication: **Predefined credential type → Zoho OAuth2 API**
  - Create/attach **Zoho OAuth2** credentials with scopes allowing Deals read (and later Tasks create).
- Method: GET
- URL:
  - `https://www.zohoapis.com/crm/v2/Deals/search?criteria=(Stage:in:Qualification,Negotiation,Proposal,Demo,Value Proposition)`
- Connect to the split node.

5) **Add Split Out node**
- Node: **Split Out**
- Field to split out: `data`
- Connect to the Code node.

6) **Add Code node to compute metrics**
- Node: **Code**
- Paste logic that computes:
  - `inactivity_days` from `Last_Activity_Time`
  - `stage_days` from `Stage_Span` or `Stage_Start_Time`
  - `deal_age` from `Created_Time`
- Ensure it returns one item per deal with the new fields merged in.
- Connect to IF node “Filter stalled Deals”.

7) **Add IF node to filter stalled deals**
- Node: **IF**
- Condition (number): `{{ $json.inactivity_days || 0 }}` **gte** `2`
- Connect the **true** output to the AI scoring chain node.
- (Leave false output unconnected or connect to logging if desired.)

8) **Add Gemini Chat Model**
- Node: **Google Gemini Chat Model** (LangChain)
- Add **Google PaLM/Gemini** credential (API key / project config as required by your n8n node).
- Keep defaults unless you want to set temperature/model explicitly.

9) **Add Structured Output Parser**
- Node: **Structured Output Parser**
- Provide the JSON structure example (fields: health_score, risk_level, top_reasons, recommended_actions, suggested_message).

10) **Add LLM Chain node (“AI Deal Health Scoring”)**
- Node: **LLM Chain** (LangChain)
- Set **Prompt type** to define (custom prompt).
- Prompt should include injected expressions for:
  - Stage, inactivity_days, stage_days, deal_age, Amount, Last_Activity_Time, Closing_Date
- Enable output parsing and connect:
  - **Gemini Chat Model → (ai_languageModel) → LLM Chain**
  - **Structured Output Parser → (ai_outputParser) → LLM Chain**
- Connect **Filter stalled Deals (true)** → **LLM Chain (main)**.

11) **Add IF node (“Is Deal At-Risk?”)**
- Node: **IF**
- Condition: `{{ $json.output.health_score }}` **lt** `60`
- Enable loose type validation (or ensure parser returns a number).
- Connect **true** output to Gmail node.

12) **Add Gmail node (“Send At-Risk Alert Email”)**
- Node: **Gmail → Send**
- Credential: **Gmail OAuth2** (ensure send scope enabled).
- To: `{{ $('Calculate Deal Metrics - Deal Inactivitu and Stage Age').item.json.Owner.email }}`
- Subject: “Deal At-Risk Alert”
- Body: include deal fields + `$json.output.*` from the AI output.
- Connect to Zoho task creation node.

13) **Add Zoho Task creation (HTTP Request)**
- Node: **HTTP Request**
- Authentication: **Zoho OAuth2 API**
- Method: POST
- URL: `https://www.zohoapis.com/crm/v2/Tasks`
- Body type: JSON
- Include task payload:
  - Due date: `{{ $now.plus({ days: 3 }).toISODate() }}`
  - Link to deal via `What_Id` using the deal `id`
  - `$se_module`: “Deals”
- Connect from **Send At-Risk Alert Email** to this node.

14) **Test safely before enabling**
- Temporarily modify Zoho search criteria to return 1 known deal (or add a limit strategy) and run manually.
- Verify:
  - `data` array exists
  - metrics are computed
  - Gemini returns parseable JSON
  - email recipient resolves to a valid address
  - Zoho task is created and linked to the right deal
- Then enable the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Zoho CRM – Deal Health Predictor (Overview) — Runs weekly, checks open deals for stall risk, computes inactivity/stage metrics, scores stalled deals with Gemini, emails the owner, creates a high-priority linked Zoho task. Setup: update Zoho/Gmail credentials, confirm stage names, confirm thresholds, test with one record, enable after sales approval. | Workflow sticky note (“Zoho CRM – Deal Health Predictor (Overview)”) |
| Confirm all stage names match Zoho exactly; otherwise the search query returns nothing. | Setup note embedded in sticky note |
| Consider replacing fallback email logic with reliable deal owner email field from Zoho (ensure `Owner.email` exists in API response). | Setup note embedded in sticky note |
| This workflow does not handle Zoho pagination; large pipelines may require looping over pages. | Derived integration note (Zoho search behavior) |