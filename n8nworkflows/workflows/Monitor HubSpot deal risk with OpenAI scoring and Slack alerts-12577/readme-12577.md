Monitor HubSpot deal risk with OpenAI scoring and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-hubspot-deal-risk-with-openai-scoring-and-slack-alerts-12577


# Monitor HubSpot deal risk with OpenAI scoring and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Monitor HubSpot deal risk with OpenAI scoring and Slack alerts

**Purpose:**  
This workflow monitors **active HubSpot deals** on a daily schedule, enriches each deal with its most recent engagement context, uses an **OpenAI-powered agent** to score conversion probability and behavioral signals, then:
- **Alerts in Slack** when a deal needs immediate attention
- **Logs the analysis** into **Google Sheets** for tracking and forecasting

**Target use cases:**
- Sales ops / RevOps pipeline monitoring
- Early detection of stalled/high-risk opportunities
- Standardized AI-based deal health scoring and suggested next actions

### Logical blocks
**1.1 Trigger + Fetch Deals (HubSpot)**  
Runs daily, pulls all deals, normalizes deal fields.

**1.2 Filter Active Deals + Per-deal Loop**  
Keeps only active (non-closed) deals and iterates through them.

**1.3 Engagement Enrichment (HubSpot via HTTP + HubSpot node)**  
Fetches engagement associations for the deal and retrieves one engagement’s details, then normalizes engagement fields.

**1.4 AI Scoring (LangChain Agent + OpenAI model)**  
Prompts the AI to generate a structured analysis.

**1.5 Parse AI Output + Alerting + Logging**  
Parses the AI text into structured fields, sends Slack alert if required, then writes results to Google Sheets; continues loop.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger + Fetch Deals (HubSpot)

**Overview:**  
Runs every day at a fixed time and retrieves HubSpot deals with selected properties. Outputs a list of deals for further formatting.

**Nodes involved:**
- Schedule Trigger
- Get Active Deals from HubSpot
- Formatting Data

#### Node: Schedule Trigger
- **Type / Role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — starts workflow on a schedule.
- **Configuration:** Triggers daily at **16:00** (4 PM) server/workflow timezone.
- **Inputs/Outputs:** Entry node → outputs to “Get Active Deals from HubSpot”.
- **Version notes:** v1.2; schedule UI/fields vary slightly across n8n versions.
- **Potential failures / edge cases:**
  - Timezone confusion (instance timezone vs user expectation).
  - If n8n is down at 16:00, run may be missed unless configured otherwise (n8n scheduling behavior depends on hosting/runtime).

#### Node: Get Active Deals from HubSpot
- **Type / Role:** `HubSpot` (`n8n-nodes-base.hubspot`) — fetches deals.
- **Configuration choices (interpreted):**
  - **Resource:** Deal
  - **Operation:** Get All
  - **Return All:** Enabled (can be large)
  - **Properties requested:** `dealname, amount, dealstage, closedate, hs_all_owner_ids, createdate, notes_last_contacted, num_associated_contacts, hs_lastmodifieddate`
  - **Auth:** HubSpot **App Token** (private app token)
- **Inputs/Outputs:** From trigger → to “Formatting Data”.
- **Version notes:** v2; HubSpot node has changed between major versions (property selection and pagination behavior can differ).
- **Potential failures / edge cases:**
  - HubSpot auth/token revoked → 401.
  - Rate limits on large portals.
  - `returnAll=true` can cause long execution times / high memory on very large pipelines.

#### Node: Formatting Data
- **Type / Role:** `Code` (`n8n-nodes-base.code`) — normalizes deal objects into a compact schema used downstream.
- **Key logic / variables:**
  - Reads `deal.properties.*.value` and derives:
    - `dealAgeDays`
    - `daysSinceActivity` (prefers `notes_last_contacted`, fallback to `hs_lastmodifieddate`, else `999`)
    - `isClosed` and `isActive` based on `dealstage` in `['closedwon','closedlost']`
  - Maps `hs_all_owner_ids` into `ownerId` (if present).
- **Inputs/Outputs:** HubSpot deals → formatted deals → to “If” (active filter).
- **Potential failures / edge cases:**
  - HubSpot deal payload shapes differ across API versions; this code expects a legacy-like `properties.{name}.value` structure.
  - Associations fields referenced (`deal.associations?.associatedVids`, `associatedCompanyIds`) may not exist depending on HubSpot API response settings; then `totalContacts/totalCompanies` remain absent.
  - Date parsing assumes millisecond timestamps in strings; if HubSpot returns ISO strings, parsing will be wrong.

**Sticky note (applies to this block):**  
“Step 1 – Trigger and collect active deals… runs everyday at 4PM… fetches all active (non-closed) deals… formats key fields…”

---

### 2.2 Filter Active Deals + Per-deal Loop

**Overview:**  
Filters to only active deals (non-closed) and processes each deal one-by-one using a batch loop.

**Nodes involved:**
- If
- Loop Over Items1

#### Node: If
- **Type / Role:** `IF` (`n8n-nodes-base.if`) — gate to keep only active deals.
- **Configuration:**
  - Condition: `{{ $json.isActive }} == true`
- **Inputs/Outputs:**
  - Input from “Formatting Data”
  - **True path** → “Loop Over Items1”
  - False path unused (no connection)
- **Potential failures / edge cases:**
  - If `isActive` missing (due to upstream format changes), strict validation may treat as invalid/false and drop deals.

#### Node: Loop Over Items1
- **Type / Role:** `Split In Batches` (`n8n-nodes-base.splitInBatches`) — iterates over deals.
- **Configuration:** Default batch settings (not explicitly set).
- **Inputs/Outputs:**
  - Receives list of active deals.
  - Its **“next batch”** behavior is used by connecting the loop-back from later nodes:
    - Main output index 1 goes to “HTTP Request” (process current item)
    - Loop continues when “Filter Alerts Needed” (false path) or “Append or update row in sheet” routes back into this node.
- **Potential failures / edge cases:**
  - If no items, loop does nothing.
  - Large volumes may exceed execution time limits depending on hosting plan.
  - Connections indicate a loopback pattern; miswiring can cause infinite loops or skipping items if altered.

---

### 2.3 Engagement Enrichment (HubSpot via HTTP + HubSpot node)

**Overview:**  
For each deal, retrieves associated engagements and fetches details of the first engagement returned, then normalizes the engagement record.

**Nodes involved:**
- HTTP Request
- Get an engagement
- Extracts Data

#### Node: HTTP Request
- **Type / Role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — calls HubSpot CRM associations endpoint.
- **Configuration choices:**
  - URL: `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.dealId }}/associations/engagements`
  - Sends headers:
    - `Content-Type: application/json`
    - `Authorization: Bearer YOUR_TOKEN_HERE` (**placeholder**)
- **Inputs/Outputs:** From “Loop Over Items1” → to “Get an engagement”.
- **Version notes:** v4.3; HTTP node options differ across versions (response formats, pagination, etc.).
- **Potential failures / edge cases:**
  - **Token mismatch risk:** workflow uses HubSpot node with appToken, but this HTTP call uses a separate Bearer token placeholder. If not replaced, it will fail.
  - HubSpot v3 associations response might not include `results[0].id` in the expected shape or may return empty results.
  - Rate limiting (429) if many deals.

#### Node: Get an engagement
- **Type / Role:** `HubSpot` (`n8n-nodes-base.hubspot`) — fetches a specific engagement object.
- **Configuration:**
  - **Resource:** Engagement
  - **Operation:** Get
  - **Engagement ID:** `{{ $json.results[0].id }}`
  - **Auth:** appToken
- **Inputs/Outputs:** From “HTTP Request” → to “Extracts Data”.
- **Potential failures / edge cases:**
  - If `results` is empty, expression `results[0].id` fails → node error.
  - Engagement IDs returned by associations may not match what this HubSpot node expects (depending on HubSpot API changes).
  - If engagement access scope missing → 403.

#### Node: Extracts Data
- **Type / Role:** `Code` — normalizes engagement payload into a flat structure for AI.
- **Key logic:**
  - Expects input items containing `json.engagement`, `json.metadata`, `json.associations`.
  - Produces fields like:
    - `dealId` from `assoc.dealIds?.[0]`
    - `timestamp` converted from seconds to ISO
    - `internalNotes` from `meta.internalMeetingNotes || meta.internalNotes`
    - Type-specific fields for MEETING/CALL/EMAIL/NOTE/TASK
    - `daysSinceEngagement`
- **Inputs/Outputs:** HubSpot engagement → normalized engagement → “AI Agent”.
- **Potential failures / edge cases:**
  - If HubSpot node output shape differs (missing `engagement/metadata/associations`), this code will throw (e.g., `eng.id` when `eng` undefined).
  - Meeting duration in the prompt uses `meetingEndTime - meetingStartTime`, but these are ISO strings; subtraction will yield `NaN` unless converted to timestamps.

**Sticky note (applies to this block):**  
“Step 2 – Enrich deals with engagement data… retrieve associated engagements… normalized and enriched…”

---

### 2.4 AI Scoring (LangChain Agent + OpenAI model)

**Overview:**  
Sends the deal and engagement context to an AI agent with strict formatting requirements, using an OpenAI chat model as the language model.

**Nodes involved:**
- AI Agent
- OpenAI Chat Model

#### Node: OpenAI Chat Model
- **Type / Role:** `OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM backend to the agent.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Built-in tools: none configured
- **Inputs/Outputs:**
  - Connected to AI Agent via `ai_languageModel` channel.
- **Potential failures / edge cases:**
  - Missing/invalid OpenAI credentials.
  - Model availability changes or permission restrictions.
  - Token limits if engagement body/internal notes are long.

#### Node: AI Agent
- **Type / Role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt and requests model output.
- **Configuration:**
  - Prompt type: “define”
  - Prompt includes:
    - Deal fields referenced from the looping node: `{{ $('Loop Over Items1').item.json.* }}`
    - Engagement fields from current item: `{{ $json.type }}`, `{{ $json.body }}`, etc.
    - Requires **EXACT format** sections (Conversion Probability, Confidence, Behavioral Score breakdown, Risk Level, signals lists, etc.)
- **Inputs/Outputs:**
  - Input: normalized engagement item
  - Output: to “Format Data1”
- **Potential failures / edge cases:**
  - Cross-node item reference `$('Loop Over Items1').item.json...` depends on correct loop context; changes in loop structure may break expressions.
  - The model may not follow the “EXACT format”, causing downstream parsing failures.
  - Sensitive data considerations: engagement bodies may contain private customer data; ensure compliance.

---

### 2.5 Parse AI Output + Alerting + Logging

**Overview:**  
Parses the AI’s structured text into machine-readable fields, flags deals needing immediate attention, sends Slack alerts, and stores results in Google Sheets.

**Nodes involved:**
- Format Data1
- Filter Alerts Needed
- Send Slack Alert
- Append or update row in sheet

#### Node: Format Data1
- **Type / Role:** `Code` — parses AI output text with regex into structured JSON.
- **Key logic:**
  - Reads AI output from `item.json.output || item.json.response || item.json.text || ''`
  - Extracts:
    - `conversionProbability` (number)
    - `riskLevel`, `dealHealthTrend`, booleans (e.g., `immediateActionRequired`)
    - lists for `positiveSignals`, `riskFactors`, `recommendedActions`
    - `predictedCloseDate` expects `YYYY-MM-DD`
    - `alertPriority` expects `P0-Critical|P1-High|P2-Medium|P3-Low`
  - Derives flags:
    - `isHighRisk` if `riskLevel` is Red/Critical
    - `needsImmediateAttention` if `immediateActionRequired`
    - `isStalled` if `dealHealthTrend === 'Stalled'`
    - `lowProbability` if probability < 50
  - Keeps `fullAnalysis` and `analysisTimestamp`
- **Inputs/Outputs:** From “AI Agent” → to “Filter Alerts Needed”.
- **Potential failures / edge cases:**
  - Regex fragility: any deviation in headings or punctuation breaks extraction.
  - `Summary:` extraction exists but the AI prompt does not require a `Summary:` section; likely `summaryText` will be null/empty.
  - If model outputs probability like “65 percent” instead of “65%”, parsing fails.

#### Node: Filter Alerts Needed
- **Type / Role:** `IF` — decides whether to send Slack alert.
- **Configuration:**
  - Condition: `{{ $json.needsImmediateAttention }} == true`
- **Inputs/Outputs:**
  - True path → “Send Slack Alert”
  - False path → loops back to “Loop Over Items1” (continue next deal)
- **Potential failures / edge cases:**
  - If parsing failed and `needsImmediateAttention` is undefined, no alerts will fire.

#### Node: Send Slack Alert
- **Type / Role:** `Slack` — posts a formatted message to a channel.
- **Configuration:**
  - Auth: OAuth2
  - Operation: send message to channel
  - Channel: `YOUR_SLACK_CHANNEL_ID` (placeholder)
  - Message: detailed Slack markdown using many fields, including mapping over arrays:
    - `riskFactors.map(...)`, `positiveSignals.map(...)`, `recommendedActions.map(...)`
- **Inputs/Outputs:** From “Filter Alerts Needed”(true) → to “Append or update row in sheet”.
- **Potential failures / edge cases:**
  - If any of `riskFactors/positiveSignals/recommendedActions` is not an array (null), `.map` will throw an expression error.
  - Slack formatting: long messages may be truncated by Slack limits.
  - Missing channel permission or invalid OAuth token.

#### Node: Append or update row in sheet
- **Type / Role:** `Google Sheets` — persists analysis results.
- **Configuration choices:**
  - Operation: **Append or Update**
  - Document: `YOUR_GOOGLE_SHEET_ID` (placeholder)
  - Sheet: `Sheet1` / `gid=0`
  - Matching column: `dealName` (updates row with same dealName)
  - Writes many columns from `$('Filter Alerts Needed').item.json.*` plus deal name from loop item.
- **Inputs/Outputs:** From “Send Slack Alert” → loops back to “Loop Over Items1”.
- **Potential failures / edge cases:**
  - Matching on `dealName` can collide if multiple deals share the same name; safer key would be `dealId`.
  - Many fields are written as strings; arrays are flattened with `[0..2]` and newline joins—missing indexes yield `undefined` strings.
  - Google Sheets API quotas and auth failures.
  - If Slack alert is not sent (false path), this node is never reached; therefore **only alerted deals are logged** (by current wiring).

**Sticky note (applies to this block):**  
“Step 3 – Analyze deals and notify team… AI to evaluate deal health… High-risk… trigger Slack alerts and are logged in Google Sheets…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Daily trigger at 16:00 | — | Get Active Deals from HubSpot | Step 1 – Trigger and collect active deals… runs everyday at 4PM… fetches all active (non-closed) deals… formats key fields… |
| Get Active Deals from HubSpot | hubspot | Pull all deals + selected properties | Schedule Trigger | Formatting Data | Step 1 – Trigger and collect active deals… runs everyday at 4PM… fetches all active (non-closed) deals… formats key fields… |
| Formatting Data | code | Normalize deals, derive activity/age flags | Get Active Deals from HubSpot | If | Step 1 – Trigger and collect active deals… runs everyday at 4PM… fetches all active (non-closed) deals… formats key fields… |
| If | if | Keep only active deals | Formatting Data | Loop Over Items1 | Step 2 – Enrich deals with engagement data… retrieve associated engagements… normalized and enriched… |
| Loop Over Items1 | splitInBatches | Iterate through deals one-by-one | If; Filter Alerts Needed (false path); Append or update row in sheet | HTTP Request (per item) | Step 2 – Enrich deals with engagement data… retrieve associated engagements… normalized and enriched… |
| HTTP Request | httpRequest | Get deal→engagement associations from HubSpot API | Loop Over Items1 | Get an engagement | Step 2 – Enrich deals with engagement data… retrieve associated engagements… normalized and enriched… |
| Get an engagement | hubspot | Fetch engagement details by ID | HTTP Request | Extracts Data | Step 2 – Enrich deals with engagement data… retrieve associated engagements… normalized and enriched… |
| Extracts Data | code | Flatten/normalize engagement fields | Get an engagement | AI Agent | Step 3 – Analyze deals and notify team… AI evaluate deal health… Slack alerts + Sheets logging… |
| OpenAI Chat Model | lmChatOpenAi | LLM backend for AI Agent (gpt-4o-mini) | — | AI Agent (ai_languageModel) | Step 3 – Analyze deals and notify team… AI evaluate deal health… Slack alerts + Sheets logging… |
| AI Agent | agent | Produce structured analysis text | Extracts Data; OpenAI Chat Model | Format Data1 | Step 3 – Analyze deals and notify team… AI evaluate deal health… Slack alerts + Sheets logging… |
| Format Data1 | code | Regex-parse AI output into fields + flags | AI Agent | Filter Alerts Needed | Step 3 – Analyze deals and notify team… AI evaluate deal health… Slack alerts + Sheets logging… |
| Filter Alerts Needed | if | Alert gate (`needsImmediateAttention`) | Format Data1 | Send Slack Alert (true); Loop Over Items1 (false) | Step 3 – Analyze deals and notify team… AI evaluate deal health… Slack alerts + Sheets logging… |
| Send Slack Alert | slack | Post alert message to Slack channel | Filter Alerts Needed (true) | Append or update row in sheet | Step 3 – Analyze deals and notify team… AI evaluate deal health… Slack alerts + Sheets logging… |
| Append or update row in sheet | googleSheets | Append/update alert results in Google Sheet | Send Slack Alert | Loop Over Items1 | Step 3 – Analyze deals and notify team… AI evaluate deal health… Slack alerts + Sheets logging… |
| Sticky Note1 | stickyNote | Comment / block label | — | — | (Contains: Step 1 description) |
| Sticky Note2 | stickyNote | Comment / block label | — | — | (Contains: Step 2 description) |
| Sticky Note3 | stickyNote | Comment / block label | — | — | (Contains: Step 3 description) |
| Sticky Note4 | stickyNote | Global workflow description + setup steps | — | — | # AI-Powered HubSpot Deal Risk Monitoring… setup steps 1–6 |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Schedule Trigger**
   - Add node: **Schedule Trigger**
   - Set an **interval** to run daily at **16:00** (4 PM).

2) **Fetch deals from HubSpot**
   - Add node: **HubSpot**
   - Resource: **Deal**
   - Operation: **Get All**
   - Return All: **true**
   - Properties: select  
     `dealname, amount, dealstage, closedate, hs_all_owner_ids, createdate, notes_last_contacted, num_associated_contacts, hs_lastmodifieddate`
   - Credentials: create **HubSpot Private App Token** credential.
   - Connect: Schedule Trigger → HubSpot (Get All deals)

3) **Normalize deal data**
   - Add node: **Code** named “Formatting Data”
   - Implement logic to:
     - Map deal fields into `dealId, dealName, amount, dealStage, createDate, closeDate, lastContactedDate, dealAgeDays, daysSinceActivity`
     - Derive `isActive` / `isClosed`
   - Connect: HubSpot → Formatting Data

4) **Filter only active deals**
   - Add node: **IF** named “If”
   - Condition: Boolean equals → `{{ $json.isActive }}` is `true`
   - Connect: Formatting Data → If (true output used)

5) **Loop over deals**
   - Add node: **Split In Batches** named “Loop Over Items1”
   - Keep defaults (batch size optional).
   - Connect: If (true) → Loop Over Items1

6) **Fetch engagement associations (HTTP)**
   - Add node: **HTTP Request**
   - Method: GET
   - URL: `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.dealId }}/associations/engagements`
   - Headers:
     - `Authorization: Bearer <YOUR_HUBSPOT_TOKEN>`
     - `Content-Type: application/json`
   - Connect: Loop Over Items1 → HTTP Request  
   - Important: use a valid token; ideally standardize by using one auth method (either use HubSpot node everywhere, or store token as credential/env var).

7) **Get engagement details**
   - Add node: **HubSpot** named “Get an engagement”
   - Resource: **Engagement**
   - Operation: **Get**
   - Engagement ID: `{{ $json.results[0].id }}`
   - Credentials: HubSpot app token
   - Connect: HTTP Request → Get an engagement

8) **Normalize engagement data**
   - Add node: **Code** named “Extracts Data”
   - Build a flat engagement object (type, timestamps, title/body/internal notes, associations, type-specific metadata).
   - Connect: Get an engagement → Extracts Data

9) **Add OpenAI Chat Model**
   - Add node: **OpenAI Chat Model**
   - Model: `gpt-4o-mini`
   - Credentials: OpenAI API key credential in n8n.
   - Do not connect on “main”; this connects to the agent via the model port.

10) **Add AI Agent**
   - Add node: **AI Agent** (LangChain Agent)
   - Prompt: paste the structured prompt requiring the EXACT format and include:
     - Deal fields via `$('Loop Over Items1').item.json.*`
     - Engagement fields via `$json.*`
   - Connect:
     - Extracts Data → AI Agent (main)
     - OpenAI Chat Model → AI Agent (ai_languageModel)

11) **Parse AI output**
   - Add node: **Code** named “Format Data1”
   - Use regex extraction to produce:
     - `conversionProbability, riskLevel, dealHealthTrend, immediateActionRequired, positiveSignals[], riskFactors[], recommendedActions[]`, etc.
   - Derive flags:
     - `needsImmediateAttention`, `isHighRisk`, `isStalled`, `lowProbability`
   - Connect: AI Agent → Format Data1

12) **Filter alerts**
   - Add node: **IF** named “Filter Alerts Needed”
   - Condition: `{{ $json.needsImmediateAttention }}` equals `true`
   - Connect: Format Data1 → Filter Alerts Needed

13) **Send Slack alert**
   - Add node: **Slack**
   - Auth: OAuth2 (connect Slack)
   - Operation: send message to a channel
   - Channel: select your channel ID
   - Message: use the provided Slack template, referencing `$json.*`
   - Connect: Filter Alerts Needed (true) → Send Slack Alert

14) **Write to Google Sheets**
   - Add node: **Google Sheets**
   - Operation: **Append or Update**
   - Document: select your spreadsheet
   - Sheet: select `Sheet1`
   - Matching column: `dealName` (as implemented)  
     - Recommended improvement when recreating: use `dealId` as the unique match key.
   - Map columns from the parsed AI data and deal name from the loop item.
   - Connect: Send Slack Alert → Google Sheets

15) **Close the loop**
   - Connect:
     - Google Sheets → Loop Over Items1 (to continue next deal)
     - Filter Alerts Needed (false) → Loop Over Items1 (to continue next deal when no alert)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| # AI-Powered HubSpot Deal Risk Monitoring — Runs on schedule, collects active deals, enriches with engagement activity, uses AI to predict conversion probability/deal health, flags risk/stagnation, alerts Slack, logs to Google Sheets. Setup: connect HubSpot/OpenAI/Slack/Google Sheets; replace example IDs; activate. | From workflow sticky note (global description). |
| Step 1 – Trigger and collect active deals (4PM schedule; fetch HubSpot deals; format key fields). | Sticky note describing block 1. |
| Step 2 – Enrich deals with engagement data (retrieve & normalize engagement context). | Sticky note describing block 2. |
| Step 3 – Analyze deals and notify team (AI scoring; Slack alerts; Google Sheets logging). | Sticky note describing block 3. |