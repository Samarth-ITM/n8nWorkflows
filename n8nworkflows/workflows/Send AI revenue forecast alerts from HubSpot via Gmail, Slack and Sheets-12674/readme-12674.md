Send AI revenue forecast alerts from HubSpot via Gmail, Slack and Sheets

https://n8nworkflows.xyz/workflows/send-ai-revenue-forecast-alerts-from-hubspot-via-gmail--slack-and-sheets-12674


# Send AI revenue forecast alerts from HubSpot via Gmail, Slack and Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send AI revenue forecast alerts from HubSpot via Gmail, Slack and Sheets

**Purpose:**  
When a new **HubSpot deal is created**, the workflow fetches the full deal pipeline, normalizes key fields, sends the structured pipeline to an **LLM (Groq)** to produce an executive-ready revenue forecast and risk analysis, then **distributes** the report through **Gmail** and **Slack**, and **logs** the results into **Google Sheets** for audit/trackability.

**Target use cases**
- Sales leadership pipeline visibility (next 30 days outlook)
- Automated risk alerts for deals likely to slip
- Consistent forecasting snapshots stored over time in Sheets

### Logical blocks
**1.1 HubSpot Intake & Pipeline Retrieval**  
Trigger on deal creation → fetch all deals from HubSpot.

**1.2 Data Standardization (pre-AI)**  
Convert HubSpot deal properties to a consistent schema (name, amount, probability, region, currency, etc.).

**1.3 AI Forecast Generation**  
Batch/loop behavior → send structured data to LangChain Agent backed by Groq chat model.

**1.4 Post-processing & Distribution + Logging**  
Parse AI markdown into structured fields → send full report to Gmail/Slack → append row to Google Sheets → wait/loop.

---

## 2. Block-by-Block Analysis

### 2.1 HubSpot Intake & Pipeline Retrieval
**Overview:**  
Starts the workflow on HubSpot deal creation, then retrieves the current pipeline (all deals) for analysis.

**Nodes involved:**
- **Sticky Note4** (documentation)
- **HubSpot Trigger1**
- **Get many deals1**

#### Sticky Note4 (Sticky Note)
- **Type/role:** Documentation overlay for “Step 1”.
- **Content (applies to nodes in this block):**  
  “Step 1 – Collect & prepare HubSpot deals. Triggers on deal creation, fetches all active deals, and formats key fields for AI analysis.”
- **Failure modes:** None (non-executable).

#### HubSpot Trigger1 (HubSpot Trigger)
- **Type/role:** Event trigger (entry point).
- **Configuration:**
  - Event: **`deal.creation`**
  - Uses HubSpot webhook subscription behind the scenes.
- **Outputs:** Emits the created deal event payload.
- **Connections:**  
  Output → **Get many deals1**
- **Version specifics:** `typeVersion: 1`
- **Edge cases / failures:**
  - HubSpot OAuth/Private App token invalid/expired.
  - Webhook not enabled/verified in HubSpot.
  - High event volume can cause missed events if n8n instance is down.
  - The workflow currently **does not use** the created deal ID to filter—every trigger causes a full pipeline pull.

#### Get many deals1 (HubSpot)
- **Type/role:** Data retrieval from HubSpot CRM.
- **Configuration:**
  - Resource: **Deal**
  - Operation: **GetAll**
  - Filters: empty (`{}`) → effectively “get all accessible deals” (not necessarily only “active” unless the HubSpot node defaults apply; no explicit filtering is configured).
- **Connections:**  
  Output → **Format Hubspot Data1**
- **Version specifics:** `typeVersion: 2.2`
- **Edge cases / failures:**
  - Pagination/large pipelines: performance/timeouts.
  - Rate limiting by HubSpot API.
  - Missing properties in returned payload (depends on HubSpot property permissions).
  - If closedate is not numeric (or absent) formatting may yield `null` as designed.

---

### 2.2 Data Standardization (pre-AI)
**Overview:**  
Transforms each HubSpot deal into a normalized, AI-friendly JSON object and assigns a currency based on region.

**Nodes involved:**
- **Format Hubspot Data1**

(Sticky Note4 still contextually applies.)

#### Format Hubspot Data1 (Code)
- **Type/role:** JavaScript transformation.
- **Key configuration choices:**
  - Maps region/country to currency via `getCurrency()`:
    - India→INR, USA/United States→USD, UK/United Kingdom→GBP, UAE→AED, default→USD.
  - Builds a standardized object per deal with keys:
    - `Deal Name`, `Customer Name`, `Deal Amount` (Number), `Currency`, `Stage`, `Close Date` (YYYY-MM-DD or null), `Probability (%)` (0–100), `Sales Owner`, `Region`.
  - `closedate` expected as a millisecond timestamp string/number; converted with `new Date(Number(...))`.
  - `hs_probability` expected 0–1; multiplied by 100 and rounded.
- **Inputs:** Deal list items from HubSpot node.
- **Outputs:** One item per deal, each in the standardized schema.
- **Connections:**  
  Output → **Loop Over Items1**
- **Version specifics:** `typeVersion: 2`
- **Edge cases / failures:**
  - If `closedate` is non-numeric, `Number()` becomes `NaN` → `new Date(NaN)` yields “Invalid Date”; current code avoids this by conditional check but still assumes numeric string when present.
  - `Deal Amount` coerces to `0` if missing; may hide data quality issues.
  - “Sales Owner” uses `hubspot_owner_id` (an ID), not a human name.

---

### 2.3 AI Forecast Generation
**Overview:**  
Iterates through incoming items via batching, sends data into a LangChain Agent prompt, and uses Groq Llama model as the language model.

**Nodes involved:**
- **Sticky Note5** (documentation)
- **Loop Over Items1**
- **AI Revenue Forecast & Risk Analysis1**
- **Groq Chat Model1**

#### Sticky Note5 (Sticky Note)
- **Type/role:** Documentation overlay for “Step 2”.
- **Content (applies to nodes in this block and next):**  
  “Step 2 – Generate & distribute AI forecast. AI generates revenue forecasts and risk insights, sends them to Slack and Email, and logs them in Google Sheets.”
- **Failure modes:** None.

#### Loop Over Items1 (Split In Batches)
- **Type/role:** Controls batching/iteration.
- **Configuration:**
  - Options empty → uses defaults (batch size default depends on n8n version; commonly 1).
- **Connections:**
  - **Main output (index 0)**: not used (empty in connections)
  - **Secondary output (index 1)** → **AI Revenue Forecast & Risk Analysis1**
  - **Input loopback:** **Wait1** → Loop Over Items1 (keeps workflow cycling)
- **Version specifics:** `typeVersion: 3`
- **Edge cases / failures:**
  - This wiring is unusual: the workflow uses output index **1** to proceed. If n8n behavior changes or if batches are misconfigured, the flow can stall.
  - Risk of unintended looping if batch completion isn’t handled as expected.
  - If the intention was “process all deals at once”, SplitInBatches is the wrong tool; it will process items in chunks.

#### AI Revenue Forecast & Risk Analysis1 (LangChain Agent)
- **Type/role:** LLM agent that generates the forecast narrative.
- **Configuration choices (interpreted):**
  - Prompt includes:
    - Current date: `{{$now.format('YYYY-MM-DD')}}`
    - Forecast period: “Next 30 days”
    - Pipeline data embedded as `{{JSON.stringify($json)}}`
  - System message defines role, output requirements, and tone.
  - Requires a connected language model: **Groq Chat Model1** connected via `ai_languageModel`.
- **Inputs:** One batch item at a time (based on SplitInBatches behavior); `$json` at this node is whatever item the batch outputs.
- **Outputs:** An item with an `output` field containing the AI text (expected markdown sections like `## Revenue Forecast Summary`, etc.).
- **Connections:**  
  Output → **Format AI response1**
- **Version specifics:** `typeVersion: 3`
- **Edge cases / failures:**
  - Token/context explosion: `JSON.stringify($json)` for a deal item is small; but if you intended to send the entire pipeline at once, current design may not do that.
  - Model may not follow the exact section headings expected by the parser (see next block).
  - API errors: rate limits, timeouts, invalid Groq credentials.

#### Groq Chat Model1 (LangChain Groq Chat Model)
- **Type/role:** LLM provider configuration for the agent.
- **Configuration:**
  - Model: **`llama-3.3-70b-versatile`**
  - No special options set.
- **Connections:**  
  `ai_languageModel` → **AI Revenue Forecast & Risk Analysis1**
- **Version specifics:** `typeVersion: 1`
- **Edge cases / failures:**
  - Missing/invalid Groq API key/credentials.
  - Model availability or provider outages.

---

### 2.4 Post-processing & Distribution + Logging
**Overview:**  
Parses the AI response (expected markdown) into structured values for Sheets and HTML for email/Slack, then sends notifications and logs the result, and finally waits before continuing the loop.

**Nodes involved:**
- **Format AI response1**
- **Send a message2** (Gmail)
- **Send a message3** (Slack)
- **Append row in sheet** (Google Sheets)
- **Wait1**

(Sticky Note5 contextually applies here.)

#### Format AI response1 (Code)
- **Type/role:** Parses and reformats AI output into:
  - `sheets_data` (flat fields for Sheets)
  - `email_data` (HTML-friendly sections + `full_report`)
  - `raw_output` (debug)
- **Key logic & variables:**
  - Reads AI output: `const aiOutput = $input.first().json.output;`
  - Extracts sections by regex on headings: `## <Section Title>`
    - Expects headings like:
      - `## Revenue Drivers Analysis`
      - `## High-Risk Deals`
  - Extracts revenue figures by regex looking for:
    - `Best Case: ₹...`
    - `Likely Case: ₹...`
    - `Worst Case: ₹...`
    - `Weighted Pipeline Value: ₹...`
  - Builds timestamp and forecast period in `Asia/Kolkata`.
  - Hard-coded recipients: `report_sent_to: 'user@example.com, user@example.com'`
  - Produces `email_data.full_report` by converting newlines to `<br>` and bold markdown to `<strong>`.
- **Inputs:** Output item from the LangChain Agent.
- **Outputs:** Single item with `json.sheets_data`, `json.email_data`, `json.raw_output`.
- **Connections:**  
  Output → **Send a message2**, **Send a message3**, **Append row in sheet** (parallel fan-out)
- **Version specifics:** `typeVersion: 2`
- **Edge cases / failures:**
  - If agent output doesn’t contain `##` headings exactly, `extractSection()` returns “Not available”.
  - Currency parsing is INR-specific (expects ₹). If the AI outputs USD/GBP/etc., extraction returns zeros.
  - “High-risk deals” parsing only captures one deal (first match) and relies on specific formatting like `**Deal Name:**`.
  - Recommendations parsing expects numbered list with `**Title**` formatting; may fail silently and return default text.
  - `toLocaleString` time zone support depends on runtime; most n8n Node.js builds support it, but some minimal builds may not.

#### Send a message2 (Gmail)
- **Type/role:** Email delivery of the AI report.
- **Configuration:**
  - To: `your-email` (placeholder)
  - Subject: `Revenue Forecast Summary Data`
  - Body: `={{ $json.email_data.full_report }}`
- **Inputs:** From Format AI response1.
- **Outputs:** Gmail send result.
- **Connections:** None forward.
- **Version specifics:** `typeVersion: 2.2`
- **Edge cases / failures:**
  - Gmail OAuth2 not configured or expired refresh token.
  - If HTML is expected, node may need “HTML mode”/options depending on n8n Gmail node behavior; current config just sets `message`.
  - Sending limits/quota.

#### Send a message3 (Slack)
- **Type/role:** Slack notification.
- **Configuration:**
  - Authentication: **oAuth2**
  - Channel selection: `select: channel` and a configured `channelId` (cached name shown as `all-n8ntest`)
  - Text: `={{ $json.email_data.full_report }}`
- **Inputs:** From Format AI response1.
- **Outputs:** Slack API response.
- **Connections:** None forward.
- **Version specifics:** `typeVersion: 2.4`
- **Edge cases / failures:**
  - OAuth scopes missing (e.g., `chat:write`).
  - ChannelId mismatch (cached values can break when moving instances).
  - Posting HTML-like content (from `full_report`) may look messy in Slack; Slack expects mrkdwn/plain text.

#### Append row in sheet (Google Sheets)
- **Type/role:** Logging forecast snapshot to Sheets.
- **Configuration:**
  - Operation: **Append**
  - Document and Sheet selected by ID (placeholders shown in cached fields).
  - Mapping: “defineBelow” with explicit column mapping:
    - Time Stamp, Forecast_Period, Expected_Revenue, Best_Case, Worst_Case, Weighted_Pipeline, High_Risk_Deals, Revenue_Drivers, Recommendations, Full_AI_Response, Report_Sent_To, Status
  - Sets `Status` to constant “Email Sent”.
  - Uses expressions like `={{ $json.sheets_data.best_case }}`
- **Inputs:** From Format AI response1.
- **Outputs:** Append result (row info).
- **Connections:** Output → **Wait1**
- **Version specifics:** `typeVersion: 4.7`
- **Edge cases / failures:**
  - Google auth issues (OAuth2) and permission to edit the sheet.
  - Column names must match exactly; otherwise values may not map.
  - Large `Full_AI_Response` can exceed cell limits (~50k chars in Google Sheets).

#### Wait1 (Wait)
- **Type/role:** Pauses workflow then continues; used here to feed back into batching loop.
- **Configuration:** None (no explicit duration or webhook resume configured in parameters).
- **Connections:** Output → **Loop Over Items1**
- **Version specifics:** `typeVersion: 1.1`
- **Edge cases / failures:**
  - As configured, behavior can be ambiguous: Wait node usually needs a “resume” condition (time-based or webhook). With empty parameters it may not resume as intended.
  - Can cause stuck executions if not properly configured.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note3 | Sticky Note | Workflow description and setup notes | — | — | ## AI Revenue Forecast Generator… (full note content below in section 5) |
| Sticky Note4 | Sticky Note | Block note: Step 1 | — | — | ## Step 1 – Collect & prepare HubSpot deals… |
| HubSpot Trigger1 | HubSpot Trigger | Entry point on deal creation | — | Get many deals1 | ## Step 1 – Collect & prepare HubSpot deals… |
| Get many deals1 | HubSpot | Fetch all deals from HubSpot | HubSpot Trigger1 | Format Hubspot Data1 | ## Step 1 – Collect & prepare HubSpot deals… |
| Format Hubspot Data1 | Code | Normalize HubSpot deal fields | Get many deals1 | Loop Over Items1 | ## Step 1 – Collect & prepare HubSpot deals… |
| Loop Over Items1 | Split In Batches | Batch/loop controller | Format Hubspot Data1, Wait1 | AI Revenue Forecast & Risk Analysis1 | ## Step 2 – Generate & distribute AI forecast… |
| Groq Chat Model1 | LangChain Groq Chat Model | LLM model provider | — | AI Revenue Forecast & Risk Analysis1 (ai_languageModel) | ## Step 2 – Generate & distribute AI forecast… |
| AI Revenue Forecast & Risk Analysis1 | LangChain Agent | Generate forecast narrative | Loop Over Items1 | Format AI response1 | ## Step 2 – Generate & distribute AI forecast… |
| Format AI response1 | Code | Parse AI output; prepare email + sheets payload | AI Revenue Forecast & Risk Analysis1 | Send a message2; Send a message3; Append row in sheet | ## Step 2 – Generate & distribute AI forecast… |
| Send a message2 | Gmail | Email report | Format AI response1 | — | ## Step 2 – Generate & distribute AI forecast… |
| Send a message3 | Slack | Slack report | Format AI response1 | — | ## Step 2 – Generate & distribute AI forecast… |
| Append row in sheet | Google Sheets | Log snapshot to Sheets | Format AI response1 | Wait1 | ## Step 2 – Generate & distribute AI forecast… |
| Wait1 | Wait | Pause/resume then continue loop | Append row in sheet | Loop Over Items1 | ## Step 2 – Generate & distribute AI forecast… |
| Sticky Note5 | Sticky Note | Block note: Step 2 | — | — | ## Step 2 – Generate & distribute AI forecast… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**  
   - Name it: “Send AI revenue forecast alerts from HubSpot via Gmail, Slack and Sheets”.

2) **Add HubSpot Trigger**
   - Node: **HubSpot Trigger**
   - Event: **Deal creation** (`deal.creation`)
   - Credentials: connect HubSpot (OAuth / private app depending on your setup).
   - This is the entry node.

3) **Add HubSpot “Get All Deals”**
   - Node: **HubSpot**
   - Resource: **Deal**
   - Operation: **Get All**
   - Leave filters empty (to match current behavior) or add filters if you truly want only active/open deals.
   - Connect: HubSpot Trigger → Get All Deals.

4) **Add “Format Hubspot Data” (Code)**
   - Node: **Code**
   - Paste logic to:
     - Map region→currency
     - Output per deal: Deal Name, Customer Name, Deal Amount (number), Currency, Stage, Close Date (YYYY-MM-DD), Probability (%), Sales Owner, Region
   - Connect: Get All Deals → Format Hubspot Data.

5) **Add “Split In Batches”**
   - Node: **Split In Batches**
   - Keep defaults (to match current JSON).
   - Connect: Format Hubspot Data → Split In Batches.

6) **Add AI Agent**
   - Node: **AI Agent** (LangChain Agent in n8n)
   - Prompt: include current date, “Next 30 days”, and embed pipeline JSON using `{{JSON.stringify($json)}}`.
   - System message: set role to revenue forecasting AI with structured sections.
   - Connect: Split In Batches → AI Agent.

7) **Add Groq Chat Model and attach to Agent**
   - Node: **Groq Chat Model**
   - Model: `llama-3.3-70b-versatile`
   - Credentials: Groq API key in n8n credentials for Groq/LangChain.
   - Connect model output to the agent’s **Language Model** input (the `ai_languageModel` connection type).

8) **Add “Format AI response” (Code)**
   - Node: **Code**
   - Implement parsing to produce:
     - `sheets_data` fields (timestamp, forecast period, expected/best/worst, weighted pipeline, risks, drivers, recommendations, full response)
     - `email_data.full_report` (HTML-ish)
   - Ensure it reads agent output from `$input.first().json.output`.
   - Connect: AI Agent → Format AI response.

9) **Add Gmail “Send Message”**
   - Node: **Gmail**
   - Operation: Send
   - To: your target recipients
   - Subject: “Revenue Forecast Summary Data”
   - Message: `{{ $json.email_data.full_report }}`
   - Credentials: Gmail OAuth2 in n8n.
   - Connect: Format AI response → Gmail.

10) **Add Slack “Send Message”**
   - Node: **Slack**
   - Authentication: OAuth2
   - Select: Channel
   - Channel: choose the destination channel
   - Text: `{{ $json.email_data.full_report }}`
   - Credentials: Slack OAuth2 with `chat:write`.
   - Connect: Format AI response → Slack.

11) **Add Google Sheets “Append Row”**
   - Node: **Google Sheets**
   - Operation: Append
   - Select Spreadsheet (Document) and Sheet tab
   - Map columns to expressions from `sheets_data`:
     - Example: `Best_Case = {{ $json.sheets_data.best_case }}`
   - Add constant `Status = Email Sent`
   - Credentials: Google OAuth2 with edit access to the sheet.
   - Connect: Format AI response → Google Sheets.

12) **Add Wait + loop back**
   - Node: **Wait**
   - Configure a proper wait condition (recommended):
     - Either “Wait for X seconds/minutes”, or
     - “Resume via webhook”
   - Connect: Google Sheets → Wait
   - Connect: Wait → Split In Batches (to continue batches)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **AI Revenue Forecast Generator**: Triggers when a deal is created, fetches all active deals, cleans fields, sends to AI, shares via Slack/Email, logs to Google Sheets. Setup: connect HubSpot, ensure deal properties, connect AI provider (Groq/LLM), connect Slack channel, connect Gmail, connect Sheets, turn on and test by creating a deal. | From Sticky Note3 |
| Step 1 – Collect & prepare HubSpot deals: trigger on deal creation, fetch deals, format key fields. | From Sticky Note4 |
| Step 2 – Generate & distribute AI forecast: AI forecast + risk insights, send to Slack/Email, log to Sheets. | From Sticky Note5 |