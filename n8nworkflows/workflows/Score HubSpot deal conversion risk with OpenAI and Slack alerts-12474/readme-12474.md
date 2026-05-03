Score HubSpot deal conversion risk with OpenAI and Slack alerts

https://n8nworkflows.xyz/workflows/score-hubspot-deal-conversion-risk-with-openai-and-slack-alerts-12474


# Score HubSpot deal conversion risk with OpenAI and Slack alerts

## 1. Workflow Overview

**Purpose:** This workflow monitors **active HubSpot deals** daily, enriches them with **recent engagement data**, uses **OpenAI (via an n8n AI Agent)** to score conversion probability and behavioral risk signals, then:
- **Sends a detailed Slack alert** for deals requiring immediate attention
- **Logs the analysis to Google Sheets** for tracking and forecasting

**Target use cases:**
- Sales leadership wanting proactive visibility into stalled/high-risk opportunities
- RevOps teams building a lightweight “deal health” early-warning system
- Forecasting support (Commit/Best Case/Pipeline/Omit recommendations)

### 1.1 Trigger & Deal Collection
Runs on a schedule, fetches all deals from HubSpot, and formats key fields and “active vs closed” flags.

### 1.2 Per-Deal Enrichment (Engagement Retrieval)
Processes deals one by one, retrieves associated engagements via HubSpot API, then fetches details for one engagement and normalizes the engagement into an analysis-friendly structure.

### 1.3 AI Scoring & Parsing
Sends deal + engagement context to OpenAI using an AI Agent, then parses the AI’s structured text output into normalized JSON fields and derived risk flags.

### 1.4 Alerting & Logging
If a deal needs immediate attention, posts a Slack alert; then appends/updates a row in Google Sheets and loops to the next deal.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Collect Active Deals

**Overview:** Starts daily at a fixed hour, pulls deals from HubSpot, and transforms HubSpot’s deal shape into a simplified object with computed metrics (age, inactivity, active/closed flags).

**Nodes involved:**
- Schedule Trigger
- Get Active Deals from HubSpot
- Formatting Data
- If

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration:** Runs **every day at 16:00** (4 PM).
- **Outputs:** One execution trigger into HubSpot fetch node.
- **Edge cases / failures:**
  - Timezone behavior depends on n8n instance timezone settings.
  - If instance is down at trigger time, execution is missed unless external scheduling is used.
- **Connections:** → Get Active Deals from HubSpot
- **Type version:** 1.2

#### Node: Get Active Deals from HubSpot
- **Type / role:** `n8n-nodes-base.hubspot` — fetches HubSpot CRM deals.
- **Configuration choices:**
  - **Resource:** Deal
  - **Operation:** Get All
  - **Return all:** Enabled
  - **Properties requested:** `dealname, amount, dealstage, closedate, hs_all_owner_ids, createdate, notes_last_contacted, num_associated_contacts, hs_lastmodifieddate`
  - **Auth:** HubSpot **App Token**
- **Inputs/Outputs:**
  - Input: schedule trigger
  - Output: list of deals to “Formatting Data”
- **Edge cases / failures:**
  - Invalid/expired app token → 401/403.
  - Large portals → long runtimes, pagination load, potential rate limiting.
  - Note: despite node name “Get Active Deals…”, this node itself does not filter out closed deals; that happens later.
- **Connections:** → Formatting Data
- **Type version:** 2

#### Node: Formatting Data
- **Type / role:** `n8n-nodes-base.code` — normalizes HubSpot deal records.
- **What it does (interpreted):**
  - Extracts key fields from `deal.properties.*.value`
  - Computes:
    - `daysSinceActivity` from `notes_last_contacted` else `hs_lastmodifieddate` else default `999`
    - `dealAgeDays`
    - `isClosed` and `isActive` based on `dealstage` being `closedwon/closedlost`
  - Builds a flatter object (`dealId`, `dealName`, `amount`, etc.)
  - Adds owner and dynamic contact/company IDs if present
- **Key variables/expressions:** Uses JS `Date.now()`, `parseInt`, `parseFloat`, optional chaining.
- **Inputs/Outputs:**
  - Input: HubSpot deal items
  - Output: simplified deal items containing `isActive` boolean used by the next IF node
- **Edge cases / failures:**
  - HubSpot API response shape mismatches (e.g., missing `properties.*.value`) can yield nulls/defaults.
  - Associations fields referenced (`deal.associations?.associatedVids`, `associatedCompanyIds`) may not exist depending on HubSpot API version/settings; in that case contact/company arrays stay empty.
- **Connections:** → If
- **Type version:** 2

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — filters to only active deals.
- **Condition:** `{{ $json.isActive }} == true`
- **Outputs:**
  - **True branch:** to Loop Over Items
  - **False branch:** unused (closed deals dropped)
- **Edge cases / failures:**
  - If `isActive` missing or not boolean, strict validation may cause unexpected false outcomes.
- **Connections:** True → Loop Over Items
- **Type version:** 2.3

---

### Block 2 — Per-Deal Loop & Engagement Enrichment

**Overview:** Iterates through active deals, calls HubSpot API to retrieve associated engagements, then fetches one engagement record and normalizes it into a compact “engagement context” object.

**Nodes involved:**
- Loop Over Items
- HTTP Request
- Get an engagement
- Extracts Data

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — processes deals iteratively.
- **Configuration:** Default batch options (batch size not explicitly set).
- **Important connection pattern:**
  - The node has two outputs:
    - Output 1: (unused)
    - **Output 2:** goes to HTTP Request (this is how this workflow is wired)
  - It also receives “continue” signals back from downstream nodes (Slack/Sheets) to proceed to the next batch/item.
- **Edge cases / failures:**
  - If downstream branches don’t return to Loop Over Items, the loop can stall after first item. Here both main paths return to Loop Over Items, so it continues.
- **Connections:**
  - Output 2 → HTTP Request
  - Receives from Filter Alerts Needed (false path) and from Append/update Sheet (after Slack path)
- **Type version:** 3

#### Node: HTTP Request
- **Type / role:** `n8n-nodes-base.httpRequest` — calls HubSpot CRM v3 associations endpoint.
- **Configuration choices:**
  - **URL:** `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.dealId }}/associations/engagements`
  - Headers:
    - `Content-Type: application/json`
    - `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder)
  - Sends headers enabled
- **Outputs:** Association results used by the next node to pick an engagement ID.
- **Edge cases / failures:**
  - **Token mismatch risk:** the workflow uses a HubSpot App Token in HubSpot nodes, but this HTTP node uses a **manual bearer token** placeholder. If not replaced, it will fail.
  - If a deal has **no engagements**, `results[0]` will be undefined in the next node and break.
  - HubSpot rate limits / 429.
- **Connections:** → Get an engagement
- **Type version:** 4.3

#### Node: Get an engagement
- **Type / role:** `n8n-nodes-base.hubspot` — retrieves a single engagement record by ID.
- **Configuration:**
  - **Resource:** Engagement
  - **Operation:** Get
  - **Engagement ID expression:** `{{ $json.results[0].id }}`
  - Auth: HubSpot App Token
- **Inputs/Outputs:**
  - Input: HTTP Request association list (expects `.results[0].id`)
  - Output: engagement object (HubSpot engagement shape)
- **Edge cases / failures:**
  - If `results` empty/missing → expression fails or returns undefined → HubSpot node error.
  - Engagements API access may vary by HubSpot account permissions.
- **Connections:** → Extracts Data
- **Type version:** 2.2

#### Node: Extracts Data
- **Type / role:** `n8n-nodes-base.code` — normalizes the HubSpot engagement into a flattened schema.
- **What it does:**
  - Reads `item.json.engagement`, `metadata`, `associations`, and attachments if present.
  - Produces:
    - core fields (engagementId, dealId, type, timestamps)
    - content fields (title, body, internalNotes, previews)
    - association arrays (contactIds, companyIds, dealIds, ownerIds, etc.)
    - type-specific fields (MEETING/CALL/EMAIL/NOTE/TASK)
    - `daysSinceEngagement`
- **Key variables/logic:**
  - Converts timestamps to ISO strings.
  - For meetings, computes start/end time ISO strings if provided.
- **Inputs/Outputs:**
  - Input: HubSpot engagement record
  - Output: normalized engagement item to AI Agent
- **Edge cases / failures:**
  - Assumes certain HubSpot engagement response structure; if HubSpot changes fields, some mapped values become empty.
  - For meeting duration in the AI prompt later, the workflow subtracts two ISO strings (see Block 3) which can yield invalid results unless converted first.
- **Connections:** → AI Agent
- **Type version:** 2

---

### Block 3 — AI Analysis & Output Parsing

**Overview:** Sends a structured prompt to the AI model combining deal context (from the loop item) and engagement context (from Extracts Data), then parses the AI’s text into JSON fields used for filtering, alerting, and logging.

**Nodes involved:**
- OpenAI Chat Model
- AI Agent
- Format Data
- Filter Alerts Needed

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model provider for the AI Agent.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - No special options set.
- **Credentials:** OpenAI API credential required.
- **Connections:** Provides the “ai_languageModel” input to AI Agent.
- **Edge cases / failures:**
  - Missing/invalid API key → auth errors.
  - Model availability changes or account restrictions.
- **Type version:** 1.3

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates the prompt and calls the connected chat model.
- **Configuration choices:**
  - Prompt is “define” style and demands an **EXACT format** with headings like:
    - `CONVERSION PROBABILITY: [X]%`
    - `BEHAVIORAL SCORE: [X]/100` with breakdown
    - signals, risks, recommended actions, next engagement, forecasting recommendation, predicted close date, alert priority
  - Uses deal fields from `$('Loop Over Items').item.json.*`
  - Uses engagement fields from current `$json.*` (output of Extracts Data)
- **Key expressions/variables:**
  - Many `$('Loop Over Items').item.json...` references to pull the *current deal* while processing engagement data.
  - Meeting duration expression: `{{ $json.meetingEndTime - $json.meetingStartTime }}` (likely problematic because these are ISO strings after normalization).
- **Inputs/Outputs:**
  - Input: normalized engagement item
  - Output: AI text (stored under `output`/`response`/`text` depending on node behavior/version)
- **Edge cases / failures:**
  - If AI deviates from the exact required format, downstream regex parsing may return null/empty arrays.
  - Large engagement bodies may hit token limits or increase cost/latency.
- **Connections:** → Format Data
- **Type version:** 3

#### Node: Format Data
- **Type / role:** `n8n-nodes-base.code` — parses AI text response into structured fields.
- **What it does:**
  - Reads `item.json.output || item.json.response || item.json.text || ''`
  - Uses regex to extract:
    - conversionProbability (number)
    - confidenceLevel + reasoning
    - behavioral score + subscores
    - risk level, deal trend, immediate action
    - positiveSignals, riskFactors, recommendedActions (lists parsed from numbered lines)
    - next engagement fields, forecasting recommendation + reasoning
    - predicted close date (expects `YYYY-MM-DD`)
    - alert priority
  - Derives flags:
    - `isHighRisk` if riskLevel is Red/Critical
    - `needsImmediateAttention` if immediateActionRequired is true
    - `isStalled` if trend is Stalled
    - `lowProbability` if conversionProbability < 50
  - Adds metadata `analysisTimestamp`
- **Inputs/Outputs:**
  - Input: AI Agent output item
  - Output: normalized analysis JSON for filtering and alerting
- **Edge cases / failures:**
  - Regex fragility: any formatting mismatch yields nulls/empties.
  - `predictedCloseDate` only matches strict `YYYY-MM-DD`; if AI outputs “March 3, 2026” it becomes null.
  - `summaryText` extractor looks for `Summary:` but the AI prompt does not require a `Summary:` section; likely always null.
- **Connections:** → Filter Alerts Needed
- **Type version:** 2

#### Node: Filter Alerts Needed
- **Type / role:** `n8n-nodes-base.if` — gates Slack alerts.
- **Condition:** `{{ $json.needsImmediateAttention }} == true`
- **Outputs:**
  - True → Send Slack Alert
  - False → Loop Over Items (continue loop without alert)
- **Edge cases / failures:**
  - If parsing failed and `needsImmediateAttention` is undefined, it won’t alert and will silently continue.
- **Connections:** True → Send Slack Alert; False → Loop Over Items
- **Type version:** 2

---

### Block 4 — Slack Alerting & Google Sheets Logging

**Overview:** For deals flagged as needing immediate attention, posts a rich Slack message with scores, signals, risks, actions, and forecasting info; then logs results in Google Sheets and continues looping.

**Nodes involved:**
- Send Slack Alert
- Append or update row in sheet

#### Node: Send Slack Alert
- **Type / role:** `n8n-nodes-base.slack` — sends a formatted Slack message.
- **Configuration:**
  - Auth: Slack OAuth2
  - Channel selection: by `channelId` (placeholder `YOUR_SLACK_CHANNEL_ID`)
  - Message text includes many interpolations and maps arrays into numbered lines:
    - `riskFactors.map(...).join('\n')`
    - `positiveSignals.map(...).join('\n')`
    - `recommendedActions.map(...).join('\n')`
- **Inputs/Outputs:**
  - Input: parsed analysis JSON (from Filter Alerts Needed true path)
  - Output: Slack API response → Google Sheets node
- **Edge cases / failures:**
  - If arrays are empty/null, `.map` may throw unless they’re always arrays (Format Data returns [] on failure, which is safe).
  - Slack OAuth scope missing for posting to channel.
  - Message length could exceed Slack limits if AI output is verbose.
- **Connections:** → Append or update row in sheet
- **Type version:** 2.2

#### Node: Append or update row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — persists the analysis.
- **Configuration choices:**
  - Operation: **Append or Update**
  - Document: placeholder `YOUR_GOOGLE_SHEET_ID`
  - Sheet tab: `gid=0` (Sheet1)
  - Matching column(s): `dealName`
  - Mapping mode: defineBelow (explicit column mapping)
  - Writes many fields from `$('Filter Alerts Needed').item.json.*`
- **Important behavior:** This node is only reached after Slack alert (i.e., only “immediate attention” deals are logged).
- **Edge cases / failures:**
  - Matching by `dealName` is not stable (renames or duplicates can overwrite wrong rows). Matching by `dealId` would be safer.
  - If riskFactors/positiveSignals/recommendedActions have fewer than 3 items, indexing `[1]`/`[2]` yields `undefined` strings in sheet.
  - Google OAuth permission/scopes and spreadsheet sharing issues.
- **Connections:** → Loop Over Items (continue loop)
- **Type version:** 4.7

---

### Sticky Notes (Documentation embedded in canvas)
- **Sticky Note:** “AI-Powered HubSpot Deal Risk Monitoring” (high-level purpose + setup)
- **Sticky Note1:** “Step 1 – Trigger and collect active deals”
- **Sticky Note2:** “Step 2 – Enrich deals with engagement data”
- **Sticky Note3:** “Step 3 – Analyze deals and notify team”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Daily trigger at 16:00 | — | Get Active Deals from HubSpot | Step 1 – Trigger and collect active deals |
| Get Active Deals from HubSpot | hubspot | Fetch deals (all) with selected properties | Schedule Trigger | Formatting Data | Step 1 – Trigger and collect active deals |
| Formatting Data | code | Normalize deal fields; compute activity/age; set isActive | Get Active Deals from HubSpot | If | Step 1 – Trigger and collect active deals |
| If | if | Filter only active deals | Formatting Data | Loop Over Items | Step 1 – Trigger and collect active deals |
| Loop Over Items | splitInBatches | Iterate deals one-by-one and control loop | If; Filter Alerts Needed (false); Append or update row in sheet | HTTP Request | Step 2 – Enrich deals with engagement data |
| HTTP Request | httpRequest | Get deal→engagement associations via HubSpot API | Loop Over Items | Get an engagement | Step 2 – Enrich deals with engagement data |
| Get an engagement | hubspot | Fetch a single engagement by ID | HTTP Request | Extracts Data | Step 2 – Enrich deals with engagement data |
| Extracts Data | code | Normalize engagement record into analysis schema | Get an engagement | AI Agent | Step 2 – Enrich deals with engagement data |
| OpenAI Chat Model | lmChatOpenAi | Provides LLM for agent | — | AI Agent (ai_languageModel) | Step 3 – Analyze deals and notify team |
| AI Agent | langchain.agent | Prompted analysis producing structured text output | Extracts Data; OpenAI Chat Model | Format Data | Step 3 – Analyze deals and notify team |
| Format Data | code | Regex-parse AI text into structured fields + derived flags | AI Agent | Filter Alerts Needed | Step 3 – Analyze deals and notify team |
| Filter Alerts Needed | if | Gate alerts based on needsImmediateAttention | Format Data | Send Slack Alert (true); Loop Over Items (false) | Step 3 – Analyze deals and notify team |
| Send Slack Alert | slack | Post detailed alert message | Filter Alerts Needed (true) | Append or update row in sheet | Step 3 – Analyze deals and notify team |
| Append or update row in sheet | googleSheets | Write analysis results to Google Sheets | Send Slack Alert | Loop Over Items | Step 3 – Analyze deals and notify team |
| Sticky Note | stickyNote | Canvas documentation | — | — | # AI-Powered HubSpot Deal Risk Monitoring… |
| Sticky Note1 | stickyNote | Canvas documentation | — | — | Step 1 – Trigger and collect active deals |
| Sticky Note2 | stickyNote | Canvas documentation | — | — | Step 2 – Enrich deals with engagement data |
| Sticky Note3 | stickyNote | Canvas documentation | — | — | Step 3 – Analyze deals and notify team |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Schedule Trigger”**
   - Node: *Schedule Trigger*
   - Set to run **daily at 16:00** (adjust timezone as needed).

2. **Add “Get Active Deals from HubSpot”**
   - Node: *HubSpot*
   - Credentials: create/connect **HubSpot App Token** credential.
   - Resource: **Deal**
   - Operation: **Get All**
   - Return All: **true**
   - Properties to request:  
     `dealname, amount, dealstage, closedate, hs_all_owner_ids, createdate, notes_last_contacted, num_associated_contacts, hs_lastmodifieddate`
   - Connect: Schedule Trigger → Get Active Deals from HubSpot

3. **Add “Formatting Data” (Code)**
   - Node: *Code*
   - Paste logic that:
     - Flattens `properties.*.value` into fields like `dealName`, `amount`, `dealStage`, etc.
     - Calculates `dealAgeDays` and `daysSinceActivity`
     - Sets `isActive` boolean (not closedwon/closedlost)
   - Connect: Get Active Deals from HubSpot → Formatting Data

4. **Add “If” to keep only active deals**
   - Node: *IF*
   - Condition (boolean equals): `{{ $json.isActive }}` equals `true`
   - Connect: Formatting Data → If

5. **Add “Loop Over Items”**
   - Node: *Split In Batches*
   - Use default settings (or set batch size explicitly if desired).
   - Connect: If (true output) → Loop Over Items

6. **Add “HTTP Request” (deal associations)**
   - Node: *HTTP Request*
   - Method: GET (default)
   - URL: `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.dealId }}/associations/engagements`
   - Headers:
     - `Content-Type: application/json`
     - `Authorization: Bearer <YOUR_HUBSPOT_PRIVATE_APP_TOKEN_OR_OAUTH_TOKEN>`
   - Connect: Loop Over Items **output 2** → HTTP Request  
   (Match the original wiring where output 2 is used.)

7. **Add “Get an engagement”**
   - Node: *HubSpot*
   - Credentials: same HubSpot App Token
   - Resource: **Engagement**
   - Operation: **Get**
   - Engagement ID: `{{ $json.results[0].id }}`
   - Connect: HTTP Request → Get an engagement

8. **Add “Extracts Data” (Code)**
   - Node: *Code*
   - Paste logic to normalize engagement fields (type, timestamps, title/body/internal notes, type-specific metadata).
   - Connect: Get an engagement → Extracts Data

9. **Add “OpenAI Chat Model”**
   - Node: *OpenAI Chat Model* (LangChain)
   - Credentials: create/connect **OpenAI API** credential.
   - Model: `gpt-4o-mini`

10. **Add “AI Agent”**
   - Node: *AI Agent* (LangChain Agent)
   - Prompt: paste the provided structured prompt (the “EXACT format” one).
   - Connect:
     - Extracts Data → AI Agent (main input)
     - OpenAI Chat Model → AI Agent (ai_languageModel input)

11. **Add “Format Data” (Code parser)**
   - Node: *Code*
   - Paste regex-based parsing logic that outputs fields like:
     `conversionProbability, behavioralScore, riskLevel, needsImmediateAttention, riskFactors[], positiveSignals[], recommendedActions[]`, etc.
   - Connect: AI Agent → Format Data

12. **Add “Filter Alerts Needed”**
   - Node: *IF*
   - Condition: `{{ $json.needsImmediateAttention }}` equals `true`
   - Connect: Format Data → Filter Alerts Needed

13. **Add “Send Slack Alert”**
   - Node: *Slack*
   - Credentials: connect **Slack OAuth2** credential with permission to post messages.
   - Operation: send message to channel (as configured in node UI).
   - Channel: select your channel (replace placeholder channel ID).
   - Message text: paste the formatted Slack message template (uses arrays map/join).
   - Connect: Filter Alerts Needed (true output) → Send Slack Alert

14. **Add “Append or update row in sheet”**
   - Node: *Google Sheets*
   - Credentials: connect **Google Sheets OAuth2** credential.
   - Operation: **Append or Update**
   - Document: choose your spreadsheet (replace placeholder ID).
   - Sheet: select tab (e.g., Sheet1)
   - Matching column: `dealName` (recommended to switch to `dealId` for reliability)
   - Map columns using expressions referencing `$('Filter Alerts Needed').item.json.*`
   - Connect: Send Slack Alert → Append or update row in sheet

15. **Close the loop**
   - Connect Filter Alerts Needed (false output) → Loop Over Items (to continue without alert)
   - Connect Append or update row in sheet → Loop Over Items (to continue after logging)

16. **Activate workflow**
   - Ensure all placeholders are replaced:
     - HubSpot bearer token in HTTP Request
     - Slack channel ID
     - Google Sheet ID
   - Turn workflow “Active”.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI-Powered HubSpot Deal Risk Monitoring” sticky note describes overall behavior and setup steps (connect HubSpot/OpenAI/Slack/Google Sheets, replace IDs, activate). | Embedded canvas documentation |
| Step notes explain the three-phase structure: (1) trigger & collect deals, (2) enrich with engagements, (3) AI analysis + Slack + Sheets. | Embedded canvas documentation |

Disclaimer (provided): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.*