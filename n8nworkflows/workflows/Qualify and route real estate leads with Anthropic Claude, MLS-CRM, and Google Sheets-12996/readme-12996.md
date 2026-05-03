Qualify and route real estate leads with Anthropic Claude, MLS/CRM, and Google Sheets

https://n8nworkflows.xyz/workflows/qualify-and-route-real-estate-leads-with-anthropic-claude--mls-crm--and-google-sheets-12996


# Qualify and route real estate leads with Anthropic Claude, MLS/CRM, and Google Sheets

## 1. Workflow Overview

**Title (provided):** Qualify and route real estate leads with Anthropic Claude, MLS/CRM, and Google Sheets  
**Workflow name (in JSON):** AI-Powered Lead Aggregation, Enrichment, and Intelligent Agent Routing System

**Purpose:**  
This workflow runs on a schedule, pulls real estate leads from two sources (MLS/portal API and CRM/email API), merges them into a single stream, enriches each lead with an Anthropic Claude agent using a structured JSON schema, then routes leads into **high** vs **standard** priority based on an AI-generated quality score. Finally, it logs routed leads into separate Google Sheets tabs for tracking.

### Logical Blocks
**1.1 Scheduled start & configuration**  
Sets runtime configuration (API endpoints, priority threshold, Google Sheet ID) and triggers execution daily.

**1.2 Multi-source lead acquisition & aggregation**  
Fetches leads in parallel from MLS/portals and CRM/email, then aggregates them.

**1.3 Per-lead AI enrichment (Anthropic Claude + structured output)**  
Splits the aggregated dataset into individual lead items and enriches each lead via an AI Agent node connected to an Anthropic chat model and a structured output parser.

**1.4 Priority routing & tracking (Google Sheets)**  
Compares AI score to a threshold, assigns agent/priority, and appends/updates each lead in the appropriate Google Sheet tab.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled start & configuration

**Overview:**  
Runs daily at a configured hour and defines key parameters (API endpoints, threshold, Google Sheet ID) used throughout the workflow.

**Nodes involved:**
- Schedule Trigger
- Workflow Configuration (Set)

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — time-based entry point.
- **Configuration (interpreted):** Runs every day at **09:00** (local n8n instance timezone).
- **Inputs / outputs:** Entry node → outputs to **Workflow Configuration**.
- **Edge cases / failures:**
  - Timezone confusion (instance timezone vs business timezone).
  - If n8n is down at trigger time, execution may be missed unless additional scheduling/queueing is implemented.
- **Version notes:** typeVersion `1.3`.

#### Node: Workflow Configuration
- **Type / role:** `Set` (`n8n-nodes-base.set`) — centralizes configuration variables.
- **Configuration (interpreted):**
  - Sets:
    - `mlsApiUrl` (placeholder string)
    - `crmApiUrl` (placeholder string)
    - `highPriorityThreshold` = **75**
    - `googleSheetId` (placeholder string)
  - **includeOtherFields = true** (passes through any existing fields).
- **Key expressions / variables produced:**  
  These fields are later referenced via `$('Workflow Configuration').first().json...`.
- **Inputs / outputs:** Receives from Schedule Trigger → outputs to both fetch nodes in parallel.
- **Edge cases / failures:**
  - If placeholders aren’t replaced with real values, downstream HTTP and Sheets nodes will fail.
  - If multiple items ever reach this node, downstream references use `.first()` (only first item’s config is used).
- **Version notes:** typeVersion `3.4`.

---

### 2.2 Multi-source lead acquisition & aggregation

**Overview:**  
Retrieves leads from two systems in parallel (MLS/portal and CRM/email). Both result sets are merged into one aggregated dataset for unified processing.

**Nodes involved:**
- Fetch Leads from MLS/Portals (HTTP Request)
- Fetch Leads from CRM/Email (HTTP Request)
- Aggregate All Leads (Aggregate)

#### Node: Fetch Leads from MLS/Portals
- **Type / role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — fetch leads from MLS/portal API.
- **Configuration (interpreted):**
  - URL: `={{ $('Workflow Configuration').first().json.mlsApiUrl }}`
  - Sends header `Authorization: <placeholder bearer token>`
  - Other options left default.
- **Inputs / outputs:** Input from Workflow Configuration → output to Aggregate All Leads.
- **Edge cases / failures:**
  - Auth failure (401/403) due to invalid/expired token.
  - API downtime/timeouts; response structure not matching what later nodes expect.
  - If response isn’t JSON or doesn’t contain a consistent leads array, aggregation/splitting may not work as intended.
- **Version notes:** typeVersion `4.3`.

#### Node: Fetch Leads from CRM/Email
- **Type / role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — fetch leads from CRM/email system API.
- **Configuration (interpreted):**
  - URL: `={{ $('Workflow Configuration').first().json.crmApiUrl }}`
  - Sends header `Authorization: <placeholder bearer token>`
- **Inputs / outputs:** Input from Workflow Configuration → output to Aggregate All Leads.
- **Edge cases / failures:**
  - Same as MLS request (auth, schema drift, timeouts).
- **Version notes:** typeVersion `4.3`.

#### Node: Aggregate All Leads
- **Type / role:** `Aggregate` (`n8n-nodes-base.aggregate`) — merges items from both sources into a single item.
- **Configuration (interpreted):**
  - Mode: **aggregateAllItemData** (collects all incoming item data into one aggregated structure).
- **Inputs / outputs:** Receives both HTTP outputs → sends one aggregated result to Split Leads for Processing.
- **Critical data-shape dependency:**  
  The next node splits on field `data`. This implies the aggregate result must contain a field named `data` that is an array of leads. If the upstream APIs don’t return data in a compatible shape, you may need mapping prior to aggregate/split.
- **Edge cases / failures:**
  - If either API returns an object rather than list items, aggregation may produce an unexpected structure.
  - If there are zero leads, downstream split/AI may run with empty input.
- **Version notes:** typeVersion `1`.

---

### 2.3 Per-lead AI enrichment (Anthropic Claude + structured output)

**Overview:**  
Transforms the aggregated dataset into one item per lead and enriches each lead using an AI agent backed by Anthropic Claude, forcing structured JSON output via a schema parser.

**Nodes involved:**
- Split Leads for Processing (Split Out)
- AI Lead Enrichment Agent (LangChain Agent)
- Anthropic Chat Model (LangChain Anthropic chat model)
- Structured Output Parser (LangChain structured parser)

#### Node: Split Leads for Processing
- **Type / role:** `Split Out` (`n8n-nodes-base.splitOut`) — converts an array field into multiple items.
- **Configuration (interpreted):**
  - Field to split: `data`
- **Inputs / outputs:** From Aggregate All Leads → outputs individual lead items to AI Lead Enrichment Agent.
- **Edge cases / failures:**
  - If `data` is missing or not an array, the split can error or produce no items.
- **Version notes:** typeVersion `1`.

#### Node: AI Lead Enrichment Agent
- **Type / role:** `AI Agent` (`@n8n/n8n-nodes-langchain.agent`) — orchestrates LLM call + structured parsing.
- **Configuration (interpreted):**
  - Prompt text: `Lead data to enrich: {{ JSON.stringify($json) }}`
  - System message instructs the model to:
    - analyze lead fields,
    - estimate demographics,
    - assess engagement/intent/urgency,
    - compute `leadQualityScore` (0–100),
    - suggest `bestFitAgentType` and `recommendedAction`,
    - return **structured JSON**.
  - `hasOutputParser = true` (expects parser to enforce schema).
- **Model / parser dependencies (connections):**
  - Uses **Anthropic Chat Model** via `ai_languageModel` connection.
  - Uses **Structured Output Parser** via `ai_outputParser` connection.
- **Inputs / outputs:** Input per lead item → outputs enriched lead item to Check Lead Priority.
- **Edge cases / failures:**
  - If LLM returns non-JSON or schema-incompatible JSON, the structured parser can fail.
  - Rate limits / cost spikes if many leads are processed at once.
  - If lead contains large/unstructured text, prompt may exceed context limits or increase latency.
- **Version notes:** typeVersion `3.1`.

#### Node: Anthropic Chat Model
- **Type / role:** `LM Chat (Anthropic)` (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) — provides Claude model to the agent.
- **Configuration (interpreted):**
  - Model: `claude-sonnet-4-5-20250929` (as selected in node)
  - Default options (no custom temperature etc. shown).
- **Credentials:** Anthropic API credential (“Anthropic account”).
- **Inputs / outputs:** Not in main flow; connected to AI Lead Enrichment Agent via `ai_languageModel`.
- **Edge cases / failures:**
  - Invalid credentials or depleted quota.
  - Model availability changes (Anthropic model deprecation/renaming).
- **Version notes:** typeVersion `1.3`.

#### Node: Structured Output Parser
- **Type / role:** `Structured Output Parser` (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces a JSON schema.
- **Configuration (interpreted):**
  - Manual JSON schema requiring (by property definition) fields like:
    - `leadQualityScore` (number)
    - `ageRange`, `incomeBracket`, `familyStatus`
    - `engagementLevel`, `purchaseIntent`, `urgency`
    - `bestFitAgentType`, `recommendedAction`
- **Inputs / outputs:** Not in main flow; connected to AI Lead Enrichment Agent via `ai_outputParser`.
- **Edge cases / failures:**
  - Model outputs strings for numeric fields (e.g., `"85"` vs `85`) can fail depending on parser strictness.
  - Missing keys may fail parsing or produce partial results depending on node behavior/version.
- **Version notes:** typeVersion `1.3`.

---

### 2.4 Priority routing & tracking (Google Sheets)

**Overview:**  
Routes each enriched lead into high vs standard priority based on `leadQualityScore`, assigns an agent, stamps routing time, and logs results to separate Google Sheets tabs.

**Nodes involved:**
- Check Lead Priority (IF)
- Route to Best-Fit Agent - High Priority (Set)
- Route to Best-Fit Agent - Standard Priority (Set)
- Track Engagement - High Priority (Google Sheets)
- Track Engagement - Standard Priority (Google Sheets)

#### Node: Check Lead Priority
- **Type / role:** `IF` (`n8n-nodes-base.if`) — branching decision.
- **Configuration (interpreted):**
  - Condition:  
    `AI Lead Enrichment Agent.leadQualityScore >= Workflow Configuration.highPriorityThreshold`
  - Threshold default: 75
- **Key expressions:**
  - Left: `={{ $('AI Lead Enrichment Agent').item.json.leadQualityScore }}`
  - Right: `={{ $('Workflow Configuration').first().json.highPriorityThreshold }}`
- **Inputs / outputs:** From AI Lead Enrichment Agent → True branch to high-priority route node; False branch to standard route node.
- **Edge cases / failures:**
  - If `leadQualityScore` is missing/null/non-numeric, comparison may behave unexpectedly under “loose” validation.
  - If the AI node output is nested differently than expected, expression may resolve to `undefined`.
- **Version notes:** typeVersion `2.3`.

#### Node: Route to Best-Fit Agent - High Priority
- **Type / role:** `Set` — assigns routing metadata for high-priority leads.
- **Configuration (interpreted):**
  - Sets:
    - `assignedAgent` = placeholder “High priority agent name/ID”
    - `priority` = `"high"`
    - `routedAt` = `={{ $now.toISO() }}`
  - includeOtherFields = true
- **Inputs / outputs:** From IF (true) → to Track Engagement - High Priority.
- **Edge cases / failures:**
  - Placeholder agent value not replaced (still logs, but routing is meaningless).
  - `$now` requires n8n’s expression runtime (should be available by default).
- **Version notes:** typeVersion `3.4`.

#### Node: Route to Best-Fit Agent - Standard Priority
- **Type / role:** `Set` — assigns routing metadata for standard leads.
- **Configuration (interpreted):**
  - Sets:
    - `assignedAgent` = placeholder “Standard priority agent name/ID”
    - `priority` = `"standard"`
    - `routedAt` = `={{ $now.toISO() }}`
- **Inputs / outputs:** From IF (false) → to Track Engagement - Standard Priority.
- **Edge cases / failures:** Same as high-priority routing node.
- **Version notes:** typeVersion `3.4`.

#### Node: Track Engagement - High Priority
- **Type / role:** `Google Sheets` (`n8n-nodes-base.googleSheets`) — logs/updates high-priority leads.
- **Configuration (interpreted):**
  - Operation: `appendOrUpdate`
  - Spreadsheet (Document ID): `={{ $('Workflow Configuration').first().json.googleSheetId }}`
  - Sheet/tab name: `"High Priority Leads"`
  - Column mapping: **auto-map input data**
- **Credentials:** Google Sheets OAuth2 credential (“Google Sheets account”).
- **Inputs / outputs:** From Route to Best-Fit Agent - High Priority → (end).
- **Edge cases / failures:**
  - Spreadsheet not shared with the OAuth user/service account.
  - Sheet name doesn’t exist (node may fail depending on behavior).
  - `appendOrUpdate` typically requires a key column or matching logic; if not configured beyond auto-map, it may behave like append-only or fail to detect duplicates depending on node defaults/version.
  - Data types/column names mismatch between incoming JSON and sheet headers.
- **Version notes:** typeVersion `4.7`.

#### Node: Track Engagement - Standard Priority
- **Type / role:** `Google Sheets` — logs/updates standard leads.
- **Configuration (interpreted):**
  - Operation: `appendOrUpdate`
  - Spreadsheet ID: from config
  - Sheet/tab name: `"Standard Priority Leads"`
  - Column mapping: auto-map
- **Credentials:** Google Sheets OAuth2 credential.
- **Inputs / outputs:** From Route to Best-Fit Agent - Standard Priority → (end).
- **Edge cases / failures:** Same as high-priority sheet node.
- **Version notes:** typeVersion `4.7`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Scheduled entry point (daily) | — | Workflow Configuration | ## Multi-Source Lead Acquisition and Aggregation — Simultaneously retrieves leads from MLS/real estate portals and CRM/email systems through parallel API calls, then aggregates all sources into unified dataset, ensuring no lead opportunities are missed |
| Workflow Configuration | Set | Defines shared config variables (URLs, threshold, sheet id) | Schedule Trigger | Fetch Leads from MLS/Portals; Fetch Leads from CRM/Email | ## Multi-Source Lead Acquisition and Aggregation — Simultaneously retrieves leads from MLS/real estate portals and CRM/email systems through parallel API calls, then aggregates all sources into unified dataset, ensuring no lead opportunities are missed |
| Fetch Leads from MLS/Portals | HTTP Request | Pull leads from MLS/portal API | Workflow Configuration | Aggregate All Leads | ## Multi-Source Lead Acquisition and Aggregation — Simultaneously retrieves leads from MLS/real estate portals and CRM/email systems through parallel API calls, then aggregates all sources into unified dataset, ensuring no lead opportunities are missed |
| Fetch Leads from CRM/Email | HTTP Request | Pull leads from CRM/email API | Workflow Configuration | Aggregate All Leads | ## Multi-Source Lead Acquisition and Aggregation — Simultaneously retrieves leads from MLS/real estate portals and CRM/email systems through parallel API calls, then aggregates all sources into unified dataset, ensuring no lead opportunities are missed |
| Aggregate All Leads | Aggregate | Merge leads from multiple sources into unified dataset | Fetch Leads from MLS/Portals; Fetch Leads from CRM/Email | Split Leads for Processing | ## Multi-Source Lead Acquisition and Aggregation — Simultaneously retrieves leads from MLS/real estate portals and CRM/email systems through parallel API calls, then aggregates all sources into unified dataset, ensuring no lead opportunities are missed |
| Split Leads for Processing | Split Out | Fan-out: one item per lead | Aggregate All Leads | AI Lead Enrichment Agent | ## AI-Powered Lead Enrichment and Qualification — Splits aggregated leads for efficient batch processing, deploys Anthropic Claude agents with structured output parsers to analyze lead attributes including property preferences, budget indicators and timeline urgency |
| AI Lead Enrichment Agent | LangChain Agent | Enrich lead + compute score + recommend action | Split Leads for Processing | Check Lead Priority | ## AI-Powered Lead Enrichment and Qualification — Splits aggregated leads for efficient batch processing, deploys Anthropic Claude agents with structured output parsers to analyze lead attributes including property preferences, budget indicators and timeline urgency |
| Anthropic Chat Model | LangChain Anthropic Chat Model | LLM provider for agent (Claude) | — (AI connection) | AI Lead Enrichment Agent (ai_languageModel) | ## AI-Powered Lead Enrichment and Qualification — Splits aggregated leads for efficient batch processing, deploys Anthropic Claude agents with structured output parsers to analyze lead attributes including property preferences, budget indicators and timeline urgency |
| Structured Output Parser | LangChain Structured Output Parser | Enforces schema on AI output | — (AI connection) | AI Lead Enrichment Agent (ai_outputParser) | ## AI-Powered Lead Enrichment and Qualification — Splits aggregated leads for efficient batch processing, deploys Anthropic Claude agents with structured output parsers to analyze lead attributes including property preferences, budget indicators and timeline urgency |
| Check Lead Priority | IF | Branch on leadQualityScore vs threshold | AI Lead Enrichment Agent | Route to Best-Fit Agent - High Priority; Route to Best-Fit Agent - Standard Priority | ## Priority-Based Agent Routing with Performance Tracking — Checks enriched lead scores against priority thresholds, routes high-priority leads to top-performing agents for immediate follow-up, standard leads to available agents, logs all engagement data to Google Sheets for conversion tracking |
| Route to Best-Fit Agent - High Priority | Set | Assign agent + stamp metadata for high priority | Check Lead Priority (true) | Track Engagement - High Priority | ## Priority-Based Agent Routing with Performance Tracking — Checks enriched lead scores against priority thresholds, routes high-priority leads to top-performing agents for immediate follow-up, standard leads to available agents, logs all engagement data to Google Sheets for conversion tracking |
| Route to Best-Fit Agent - Standard Priority | Set | Assign agent + stamp metadata for standard | Check Lead Priority (false) | Track Engagement - Standard Priority | ## Priority-Based Agent Routing with Performance Tracking — Checks enriched lead scores against priority thresholds, routes high-priority leads to top-performing agents for immediate follow-up, standard leads to available agents, logs all engagement data to Google Sheets for conversion tracking |
| Track Engagement - High Priority | Google Sheets | Append/update high-priority leads in sheet | Route to Best-Fit Agent - High Priority | — | ## Priority-Based Agent Routing with Performance Tracking — Checks enriched lead scores against priority thresholds, routes high-priority leads to top-performing agents for immediate follow-up, standard leads to available agents, logs all engagement data to Google Sheets for conversion tracking |
| Track Engagement - Standard Priority | Google Sheets | Append/update standard leads in sheet | Route to Best-Fit Agent - Standard Priority | — | ## Priority-Based Agent Routing with Performance Tracking — Checks enriched lead scores against priority thresholds, routes high-priority leads to top-performing agents for immediate follow-up, standard leads to available agents, logs all engagement data to Google Sheets for conversion tracking |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Schedule Trigger”**
   - Node type: *Schedule Trigger*
   - Configure: run **daily at 09:00** (set the hour in the interval rule).
   - This is the workflow entry.

2. **Create “Workflow Configuration” (Set)**
   - Node type: *Set*
   - Add fields:
     - `mlsApiUrl` (string): your MLS/portal leads endpoint
     - `crmApiUrl` (string): your CRM/email leads endpoint
     - `highPriorityThreshold` (number): `75` (or your desired threshold)
     - `googleSheetId` (string): spreadsheet ID used for tracking
   - Enable **Include Other Fields**.

3. **Connect:** Schedule Trigger → Workflow Configuration

4. **Create “Fetch Leads from MLS/Portals” (HTTP Request)**
   - Node type: *HTTP Request*
   - URL: expression referencing config  
     `$('Workflow Configuration').first().json.mlsApiUrl`
   - Add header: `Authorization: Bearer <YOUR_MLS_TOKEN>` (or whatever your API requires)
   - Keep response as JSON (default behavior usually works if API returns JSON).

5. **Create “Fetch Leads from CRM/Email” (HTTP Request)**
   - Node type: *HTTP Request*
   - URL:  
     `$('Workflow Configuration').first().json.crmApiUrl`
   - Add header: `Authorization: Bearer <YOUR_CRM_TOKEN>`

6. **Connect:** Workflow Configuration → Fetch Leads from MLS/Portals  
7. **Connect:** Workflow Configuration → Fetch Leads from CRM/Email  
   (parallel branches)

8. **Create “Aggregate All Leads” (Aggregate)**
   - Node type: *Aggregate*
   - Operation/mode: **Aggregate All Item Data** (aggregateAllItemData)

9. **Connect:** Fetch Leads from MLS/Portals → Aggregate All Leads  
10. **Connect:** Fetch Leads from CRM/Email → Aggregate All Leads

11. **Create “Split Leads for Processing” (Split Out)**
   - Node type: *Split Out*
   - Field to split out: `data`  
     (Ensure your aggregated result contains `data: [ ...leads ]`. If not, add mapping nodes to normalize both source responses into a shared `data` array.)

12. **Connect:** Aggregate All Leads → Split Leads for Processing

13. **Create “Anthropic Chat Model”**
   - Node type: *Anthropic Chat Model* (LangChain)
   - Select model: `claude-sonnet-4-5-20250929` (or closest available in your instance)
   - Create/connect **Anthropic API credentials** in n8n (API key in Credentials).

14. **Create “Structured Output Parser”**
   - Node type: *Structured Output Parser* (LangChain)
   - Schema type: *Manual*
   - Define a schema with fields:
     - `leadQualityScore` (number)
     - `ageRange`, `incomeBracket`, `familyStatus` (string)
     - `engagementLevel`, `purchaseIntent`, `urgency` (string)
     - `bestFitAgentType`, `recommendedAction` (string)

15. **Create “AI Lead Enrichment Agent”**
   - Node type: *AI Agent* (LangChain)
   - Prompt text:  
     `Lead data to enrich: {{ JSON.stringify($json) }}`
   - System message: instruct enrichment/scoring and require structured JSON output.
   - Ensure the agent is configured to use an output parser.

16. **Connect AI dependencies (non-main connections):**
   - Anthropic Chat Model → AI Lead Enrichment Agent via **ai_languageModel**
   - Structured Output Parser → AI Lead Enrichment Agent via **ai_outputParser**

17. **Connect main flow:** Split Leads for Processing → AI Lead Enrichment Agent

18. **Create “Check Lead Priority” (IF)**
   - Node type: *IF*
   - Condition: Number `>=`
   - Left value:  
     `$('AI Lead Enrichment Agent').item.json.leadQualityScore`
   - Right value:  
     `$('Workflow Configuration').first().json.highPriorityThreshold`

19. **Connect:** AI Lead Enrichment Agent → Check Lead Priority

20. **Create “Route to Best-Fit Agent - High Priority” (Set)**
   - Node type: *Set*
   - Fields:
     - `assignedAgent` (string): your high-priority agent name/ID
     - `priority` (string): `high`
     - `routedAt` (string expression): `$now.toISO()`
   - Include other fields = true

21. **Create “Route to Best-Fit Agent - Standard Priority” (Set)**
   - Node type: *Set*
   - Fields:
     - `assignedAgent` (string): your standard agent name/ID
     - `priority` (string): `standard`
     - `routedAt`: `$now.toISO()`
   - Include other fields = true

22. **Connect branches:**
   - Check Lead Priority (true) → Route to Best-Fit Agent - High Priority
   - Check Lead Priority (false) → Route to Best-Fit Agent - Standard Priority

23. **Create “Track Engagement - High Priority” (Google Sheets)**
   - Node type: *Google Sheets*
   - Credentials: configure **Google Sheets OAuth2** credential in n8n
   - Operation: `appendOrUpdate`
   - Document ID:  
     `$('Workflow Configuration').first().json.googleSheetId`
   - Sheet name: `High Priority Leads`
   - Column mapping: auto-map (ensure sheet headers match your JSON keys)

24. **Create “Track Engagement - Standard Priority” (Google Sheets)**
   - Same as above but sheet name: `Standard Priority Leads`

25. **Connect:**
   - Route to Best-Fit Agent - High Priority → Track Engagement - High Priority
   - Route to Best-Fit Agent - Standard Priority → Track Engagement - Standard Priority

26. **Prepare Google Sheet**
   - Create the spreadsheet with two tabs:
     - “High Priority Leads”
     - “Standard Priority Leads”
   - Add headers matching the fields you expect to log (including enriched fields and routing fields).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **How It Works:** Automates lead qualification and routing by enriching leads from multiple sources with AI-powered analysis, then routing to appropriate agents based on priority; logs to Google Sheets for monitoring and optimization. | Sticky note (workflow description block) |
| **Prerequisites:** Active Anthropic API account, MLS/portal API access, CRM API integration. **Use cases:** inbound portal leads, open house inquiries, email campaign scoring. **Customization:** adjust prompts/scoring. **Benefits:** reduces response time by 75%, prioritizes high-value prospects. | Sticky note (prereqs/benefits) |
| **Setup Steps:** configure schedule, API credentials, Anthropic credentials, structured parser, threshold rules, routing logic. | Sticky note (setup steps) |
| **AI-Powered Lead Enrichment and Qualification:** uses Claude + structured output to analyze preferences, budget indicators, urgency. | Sticky note (AI block rationale) |
| **Priority-Based Agent Routing with Performance Tracking:** thresholds determine routing tier; Sheets logging for conversion tracking. | Sticky note (routing/tracking rationale) |
| **Multi-Source Lead Acquisition and Aggregation:** parallel retrieval + unified dataset. | Sticky note (acquisition/aggregation rationale) |