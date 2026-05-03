Score and route real estate leads with GPT‑4.1, MLS/CRM data, and Slack alerts

https://n8nworkflows.xyz/workflows/score-and-route-real-estate-leads-with-gpt-4-1--mls-crm-data--and-slack-alerts-12995


# Score and route real estate leads with GPT‑4.1, MLS/CRM data, and Slack alerts

## 1. Workflow Overview

**Title:** Score and route real estate leads with GPT‑4.1, MLS/CRM data, and Slack alerts  
**Workflow name (internal):** AI-Powered Lead Aggregation, Enrichment, and Intelligent Agent Routing System

**Purpose:**  
This workflow periodically pulls real estate leads from multiple sources (MLS/portals + CRM/email/social), enriches each lead using an LLM, performs sentiment analysis and lead scoring, then routes the lead to the best agent using an LLM that can consult property data and historical lead context. High-priority leads trigger Slack + email notifications. All routed leads are tracked, embedded into an in-memory vector store, and stored in Postgres. A daily aggregate stats step is included.

### 1.1 Scheduling & Configuration
Runs on a schedule and sets all environment/config variables (API URLs, thresholds, Slack channel, DB table, sender email).

### 1.2 Lead Ingestion (MLS + CRM) and Merge
Fetches lead lists from two HTTP endpoints and merges them into a single stream.

### 1.3 Lead Enrichment (LLM + Structured JSON)
Uses an LLM agent to add demographic/behavioral/social enrichment and enforces a structured output schema.

### 1.4 Sentiment Gate
Runs sentiment analysis and (intended) gating on “Negative” or “Urgent” sentiment before scoring.

### 1.5 Lead Scoring (LLM + Structured JSON)
Computes conversion score (0–100), priority level, and indicators/factors.

### 1.6 Agent Routing (LLM Agent with Tools + Structured JSON)
Routes leads to best-fit agent; the routing agent can query a lead-history vector store and a Property Data HTTP tool.

### 1.7 Priority Handling & Notifications
If conversion score ≥ configured threshold → high-priority branch (Slack + email + metadata), else standard branch (metadata only).

### 1.8 Tracking, Knowledge Base Insert, Persistence, Looping, Aggregation
Adds engagement tracking fields, inserts the final lead record into an in-memory vector store (with OpenAI embeddings + document loader/splitting), stores in Postgres, loops over batches, and aggregates daily stats.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Workflow Configuration
**Overview:** Triggers the workflow daily and centralizes configuration values used by later nodes (URLs, thresholds, channel IDs, table names).  
**Nodes involved:** `Schedule Trigger`, `Workflow Configuration`

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger — entry point.
- **Config choices:** Runs at a specified hour (`triggerAtHour: 9`). Interval-based schedule.
- **Inputs/outputs:** No input. Outputs to `Workflow Configuration`.
- **Version notes:** typeVersion 1.3.
- **Failure modes:** Instance timezone mismatch; schedule misconfiguration; workflow inactive (workflow is currently `active: false`).

#### Node: Workflow Configuration
- **Type / role:** Set — defines constants and placeholders.
- **Key fields created:**
  - `mlsApiUrl`, `crmApiUrl`, `propertyApiUrl`, `agentApiUrl`
  - `highPriorityThreshold` (default 75)
  - `enrichmentApiKey` (not actually referenced elsewhere in this workflow)
  - `slackChannel`, `databaseTable`, `senderEmail`
- **Expressions/variables:** Referenced later via `$('Workflow Configuration').first().json.<key>`.
- **Inputs/outputs:** From trigger → outputs to both fetch nodes.
- **Version notes:** typeVersion 3.4.
- **Failure modes:** Placeholder values not replaced; inconsistent config keys vs later expressions.

---

### Block 2 — Lead Ingestion from MLS and CRM + Merge
**Overview:** Pulls lead data from two systems and merges streams into one combined list.  
**Nodes involved:** `Fetch Leads from MLS/Portals`, `Fetch Leads from CRM/Email/Social`, `Merge All Lead Sources`

#### Node: Fetch Leads from MLS/Portals
- **Type / role:** HTTP Request — fetch MLS/portal leads.
- **Configuration choices:**
  - `url` from config: `={{ $('Workflow Configuration').first().json.mlsApiUrl }}`
  - Response format JSON.
  - Header-auth via generic credential type; `Authorization` header placeholder.
- **Inputs/outputs:** Input from `Workflow Configuration`; output to `Merge All Lead Sources` (input index 0).
- **Version notes:** typeVersion 4.3.
- **Failure modes:** 401/403 auth; invalid JSON response; rate limiting; network timeouts; URL placeholder not replaced.

#### Node: Fetch Leads from CRM/Email/Social
- **Type / role:** HTTP Request — fetch CRM/email/social leads.
- **Configuration choices:** Same pattern as MLS node, using `crmApiUrl`.
- **Inputs/outputs:** Input from `Workflow Configuration`; output to `Merge All Lead Sources` (input index 1).
- **Version notes:** typeVersion 4.3.
- **Failure modes:** Same as above.

#### Node: Merge All Lead Sources
- **Type / role:** Merge — combines two input streams.
- **Configuration choices:** `mode: combine`, `combineByPosition`.
  - This assumes both sources return arrays of same length or compatible ordering. If lengths differ, items may mismatch.
- **Inputs/outputs:** Two inputs from both HTTP nodes → outputs to `Lead Enrichment Agent` and also to `Loop Over Leads`.
- **Version notes:** typeVersion 3.2.
- **Failure modes:** Misaligned items causing wrong lead merges; one source returns a single object while other returns an array; empty input from one side.

---

### Block 3 — Lead Enrichment (LLM Agent + Structured Output)
**Overview:** Enriches each lead with inferred demographics, behavioral scoring, and social indicators; enforces a JSON schema.  
**Nodes involved:** `Lead Enrichment Agent`, `OpenAI Model - Enrichment`, `Structured Output - Enriched Lead Data`

#### Node: Lead Enrichment Agent
- **Type / role:** LangChain Agent — LLM-driven enrichment step.
- **Configuration choices:**
  - Input text: `={{ $json }}` (the merged lead object).
  - System message instructs enrichment tasks and requests “structured JSON format”.
  - `hasOutputParser: true` so output must conform to the attached structured parser.
- **Inputs/outputs:**
  - Main input from `Merge All Lead Sources`.
  - Outputs to `Lead Scoring Agent` and `Lead Sentiment Analysis`.
  - Uses:
    - `OpenAI Model - Enrichment` via `ai_languageModel`
    - `Structured Output - Enriched Lead Data` via `ai_outputParser`
- **Version notes:** typeVersion 3.1.
- **Failure modes:** Model returns invalid JSON; missing fields required by schema; token limits if `$json` is large; hallucinated types (e.g., boolean vs string).

#### Node: OpenAI Model - Enrichment
- **Type / role:** OpenAI Chat Model — language model backend.
- **Configuration choices:** `gpt-4.1-mini`.
- **Credentials:** OpenAI credential required.
- **Connections:** Feeds `Lead Enrichment Agent` (`ai_languageModel`).
- **Version notes:** typeVersion 1.3.
- **Failure modes:** Invalid API key; quota exceeded; model not available; latency/timeouts.

#### Node: Structured Output - Enriched Lead Data
- **Type / role:** Structured Output Parser — validates/coerces LLM output to schema.
- **Schema highlights:**
  - Top-level: `leadId`, `name`, `email`, `phone`
  - Nested: `demographics` (ageRange, incomeBracket, familyStatus)
  - Nested: `socialPresence` (linkedin/facebook/instagram booleans)
  - Nested: `behavioralData` (emailEngagement, websiteVisits, propertyViews numbers)
- **Connections:** Feeds `Lead Enrichment Agent` (`ai_outputParser`).
- **Version notes:** typeVersion 1.3.
- **Failure modes:** LLM output doesn’t match schema; missing nested objects; nulls where numbers expected.

---

### Block 4 — Sentiment Analysis & Gate
**Overview:** Extracts sentiment category and checks for “Negative” or “Urgent” to gate next steps.  
**Nodes involved:** `Lead Sentiment Analysis`, `OpenAI Model - Sentiment`, `Check Sentiment Score`

#### Node: Lead Sentiment Analysis
- **Type / role:** Sentiment Analysis (LangChain) — classify text.
- **Configuration choices:**
  - Categories: `Positive, Neutral, Negative, Urgent`
  - Input text: `={{ $json.email }} {{ $json.phone }} {{ $json.name }}`
    - Note: this is not typical sentiment content; it’s mostly identifiers, so classification may be meaningless.
- **Inputs/outputs:** From `Lead Enrichment Agent` → to `Check Sentiment Score`.
- **Model connection:** Uses `OpenAI Model - Sentiment` (`ai_languageModel`).
- **Version notes:** typeVersion 1.1.
- **Failure modes:** Missing `email/phone/name` fields → empty text; poor classification; model failures.

#### Node: OpenAI Model - Sentiment
- **Type / role:** OpenAI Chat Model for sentiment node.
- **Configuration:** `gpt-4.1-mini`.
- **Credentials:** OpenAI required.
- **Connections:** `ai_languageModel` into `Lead Sentiment Analysis`.
- **Version notes:** typeVersion 1.3.
- **Failure modes:** Same OpenAI-related issues.

#### Node: Check Sentiment Score
- **Type / role:** IF — checks sentiment category.
- **Configuration choices:**
  - OR condition: sentimentCategory equals `Negative` OR `Urgent`
  - Uses expression: `{{ $('Lead Sentiment Analysis').item.json.sentimentCategory }}`
- **Inputs/outputs:** From `Lead Sentiment Analysis` → **only True branch is connected** to `Lead Scoring Agent`.
- **Important logic implication (edge case):**
  - If sentiment is **not** Negative/Urgent, the False branch goes nowhere → the lead stops here on that path.
  - Meanwhile, `Lead Enrichment Agent` also directly connects to `Lead Scoring Agent`, so scoring may occur regardless, creating **duplicate/parallel scoring** for some items (depending on execution order and item flow).
- **Version notes:** typeVersion 2.3.
- **Failure modes:** Sentiment field missing; string mismatch due to casing; unintended gating/duplication.

---

### Block 5 — Lead Scoring (LLM Agent + Structured Output)
**Overview:** Produces a conversion score, priority level, and indicators/factors from enriched lead data.  
**Nodes involved:** `Lead Scoring Agent`, `OpenAI Model - Scoring`, `Structured Output - Lead Score`

#### Node: Lead Scoring Agent
- **Type / role:** LangChain Agent — scoring logic.
- **Configuration choices:**
  - Input text: `={{ $json }}`
  - System message defines scoring rubric and requires JSON output.
  - `hasOutputParser: true`.
- **Inputs/outputs:** Receives from `Lead Enrichment Agent` (and also from `Check Sentiment Score` True branch) → outputs to `Agent Routing Agent`.
- **Model/parser:** Uses `OpenAI Model - Scoring` and `Structured Output - Lead Score`.
- **Version notes:** typeVersion 3.1.
- **Failure modes:** Invalid structured output; `conversionScore` not numeric; enum mismatch for `priorityLevel`.

#### Node: OpenAI Model - Scoring
- **Type / role:** OpenAI Chat Model.
- **Configuration:** `gpt-4.1-mini`.
- **Connections:** `ai_languageModel` to `Lead Scoring Agent`.
- **Version notes:** typeVersion 1.3.

#### Node: Structured Output - Lead Score
- **Type / role:** Structured Output Parser.
- **Schema highlights:**
  - `leadId` string
  - `conversionScore` number
  - `priorityLevel` enum: High/Medium/Low
  - `conversionIndicators` array of strings
  - `scoringFactors` object with numeric sub-scores
- **Connections:** `ai_outputParser` to `Lead Scoring Agent`.
- **Version notes:** typeVersion 1.3.

---

### Block 6 — Agent Routing (LLM Agent with Tools + Structured Output) + Agent Performance Merge
**Overview:** Determines best agent assignment and rationale; additionally fetches agent performance data and injects it into the lead record (but the routing agent does not actually wait for it in this design).  
**Nodes involved:** `Agent Routing Agent`, `OpenAI Model - Routing`, `Structured Output - Routing Decision`, `Lead Knowledge Base - Retrieve`, `Embeddings OpenAI - Retrieve`, `HTTP Request Tool - Property Data`, `Fetch Agent Performance Data`, `Merge Lead with Agent Data`

#### Node: Agent Routing Agent
- **Type / role:** LangChain Agent — assignment decisioning.
- **Configuration choices:**
  - Input text: `={{ $json }}`
  - System message: match lead to best agent by specialization, metrics, workload, territory; provide rationale; output structured JSON.
  - `hasOutputParser: true`.
- **Tools available (connected as `ai_tool`):**
  - `Lead Knowledge Base - Retrieve` (vector store retrieval tool)
  - `HTTP Request Tool - Property Data` (HTTP tool)
- **Inputs/outputs:**
  - Input from `Lead Scoring Agent`.
  - Outputs to:
    - `High-Priority Lead Filter`
    - `Fetch Agent Performance Data` (parallel branch)
- **Version notes:** typeVersion 3.1.
- **Failure modes:** Tool failures; non-deterministic routing; missing agent emails (later used by Email node).

#### Node: OpenAI Model - Routing
- **Type / role:** OpenAI Chat Model.
- **Configuration:** `gpt-4.1-mini`.
- **Connections:** `ai_languageModel` to `Agent Routing Agent`.
- **Version notes:** typeVersion 1.3.

#### Node: Structured Output - Routing Decision
- **Type / role:** Structured Output Parser.
- **Schema highlights:**
  - `leadId`
  - `assignedAgent` object: agentId, agentName, specialization, matchScore
  - `routingRationale` string
  - `alternativeAgents` array
- **Connections:** `ai_outputParser` to `Agent Routing Agent`.
- **Version notes:** typeVersion 1.3.
- **Edge case:** Later email node uses `assignedAgent.agentEmail`, but the schema does **not** define `agentEmail`; unless the LLM adds it anyway and parser permits additionalProperties, email sending may fail.

#### Node: Lead Knowledge Base - Retrieve
- **Type / role:** Vector Store (In-Memory) — retrieval tool for the routing agent.
- **Configuration:** `mode: retrieve-as-tool`, `memoryKey: lead_history`, tool description provided.
- **Connections:** Provides an `ai_tool` to `Agent Routing Agent`.
- **Version notes:** typeVersion 1.3.
- **Failure modes:** Empty memory on first run; irrelevant retrieval due to poor document formatting; memory not persisted across workflow executions (in-memory).

#### Node: Embeddings OpenAI - Retrieve
- **Type / role:** OpenAI embeddings provider for retrieval.
- **Connections:** `ai_embedding` to `Lead Knowledge Base - Retrieve`.
- **Credentials:** OpenAI required.
- **Version notes:** typeVersion 1.2.

#### Node: HTTP Request Tool - Property Data
- **Type / role:** HTTP Request Tool — callable tool from routing agent.
- **Configuration:** URL is a placeholder (hardcoded, not using `Workflow Configuration` value).
- **Connections:** `ai_tool` to `Agent Routing Agent`.
- **Version notes:** typeVersion 4.3.
- **Failure modes:** Placeholder not replaced; auth not configured; tool returns non-JSON; timeouts.

#### Node: Fetch Agent Performance Data
- **Type / role:** HTTP Request — fetch agent metrics dataset.
- **Configuration:** URL placeholder; header auth bearer placeholder.
- **Connections:** Triggered in parallel from `Agent Routing Agent` → outputs to `Merge Lead with Agent Data`.
- **Version notes:** typeVersion 4.3.
- **Failure modes:** Same HTTP/auth issues; mismatch between lead and agent dataset; slow endpoint.

#### Node: Merge Lead with Agent Data
- **Type / role:** Set — attaches fetched agent performance payload to the current item.
- **Configuration:** sets `agentPerformanceData` to `={{ $json }}` (string type but assigns an object; may coerce unexpectedly).
- **Connections:** Outputs to `High-Priority Lead Filter`.
- **Version notes:** typeVersion 3.4.
- **Important logic implication:**
  - `High-Priority Lead Filter` receives input from **two paths**:
    1) directly from `Agent Routing Agent` (routing decision)
    2) from `Merge Lead with Agent Data` (agent performance fetch result)
  - There is no explicit merge/join between these; you may end up evaluating the priority filter on the **wrong payload** (agent performance response lacking `conversionScore`) or duplicate processing.

---

### Block 7 — Priority Filter, Notifications, and Standard Handling
**Overview:** Splits leads into high-priority vs standard based on conversion score threshold; high-priority leads trigger Slack and email.  
**Nodes involved:** `High-Priority Lead Filter`, `Prepare High-Priority Lead Data`, `Notify Slack - High Priority`, `Send Email - Agent Assignment`, `Prepare Standard Lead Data`

#### Node: High-Priority Lead Filter
- **Type / role:** IF — threshold gate.
- **Configuration:**
  - Condition: `{{ $json.conversionScore }} >= {{ $('Workflow Configuration').first().json.highPriorityThreshold }}`
- **Inputs/outputs:** Receives from routing and/or agent-data path → True to `Prepare High-Priority Lead Data`, False to `Prepare Standard Lead Data`.
- **Version notes:** typeVersion 2.3.
- **Failure modes:** `conversionScore` missing/non-numeric; config threshold missing; misrouted inputs (see Block 6).

#### Node: Prepare High-Priority Lead Data
- **Type / role:** Set — adds metadata for urgent handling.
- **Adds/overrides:**
  - `priority: HIGH`
  - `notificationUrgency: immediate`
  - `followUpWindow: 24 hours`
  - `assignmentTimestamp: {{ $now.toISO() }}`
- **Outputs:** To `Track Engagement Metrics`, `Notify Slack - High Priority`, `Send Email - Agent Assignment` (three parallel branches).
- **Version notes:** typeVersion 3.4.

#### Node: Notify Slack - High Priority
- **Type / role:** Slack — channel message.
- **Configuration:**
  - Posts message including `name`, `conversionScore`, `assignedAgent.agentName`, `routingRationale`
  - `channelId` placeholder (also present in config but node uses its own placeholder field)
  - Auth: OAuth2 Slack
- **Credentials:** Slack OAuth2 required.
- **Outputs:** Continues to `Track Engagement Metrics`.
- **Version notes:** typeVersion 2.4.
- **Failure modes:** Invalid channel ID; missing scopes; rate limits; message fields missing due to schema mismatch.

#### Node: Send Email - Agent Assignment
- **Type / role:** Email Send — email agent assignment.
- **Configuration:**
  - To: `={{ $json.assignedAgent.agentEmail }}`
  - From: placeholder (should likely use config `senderEmail`, but it’s hardcoded placeholder)
  - Subject/text interpolates lead fields and follow-up window.
- **Version notes:** typeVersion 2.1.
- **Failure modes (high likelihood):**
  - `assignedAgent.agentEmail` not present in routing schema/output → expression resolves to empty → send failure.
  - Email service not configured in n8n; SMTP errors; spam policies.

#### Node: Prepare Standard Lead Data
- **Type / role:** Set — metadata for non-urgent leads.
- **Adds/overrides:**
  - `priority: STANDARD`
  - `notificationUrgency: normal`
  - `followUpWindow: 72 hours`
  - `assignmentTimestamp: {{ $now.toISO() }}`
- **Outputs:** To `Track Engagement Metrics`.
- **Version notes:** typeVersion 3.4.

---

### Block 8 — Tracking, Vector Insert, DB Storage, Looping, Aggregation
**Overview:** Adds engagement tracking fields, inserts lead into an in-memory knowledge base with embeddings + document handling, stores it in Postgres, loops in batches, and aggregates stats.  
**Nodes involved:** `Track Engagement Metrics`, `Lead Knowledge Base - Insert`, `Embeddings OpenAI - Insert`, `Document Loader - Lead Data`, `Text Splitter - Recursive`, `Store Lead in Database`, `Loop Over Leads`, `Aggregate Daily Lead Stats`

#### Node: Track Engagement Metrics
- **Type / role:** Set — adds operational metrics and ROI tracking.
- **Adds/overrides:**
  - `touchpointCount: 1`
  - `lastEngagementDate: {{ $now.toISO() }}`
  - `engagementChannel: lead_aggregation`
  - `workflowStage: routed_to_agent`
  - `roiTrackingId: {{ $('Lead Enrichment Agent').item.json.leadId }}_{{ $now.toMillis() }}`
- **Inputs/outputs:** From high-priority branches (Slack/email also feed into this) and standard branch → to `Lead Knowledge Base - Insert`.
- **Version notes:** typeVersion 3.4.
- **Failure modes:** `$('Lead Enrichment Agent').item.json.leadId` may not be in scope for items arriving from other branches → expression error or wrong leadId.

#### Node: Lead Knowledge Base - Insert
- **Type / role:** Vector Store (In-Memory) — insert documents.
- **Configuration:** `mode: insert`, `memoryKey: lead_history`.
- **Requires:** embeddings + document loader connections.
- **Inputs/outputs:** From `Track Engagement Metrics` → to `Store Lead in Database`.
- **Version notes:** typeVersion 1.3.
- **Failure modes:** In-memory store resets between executions; large documents; embedding failures.

#### Node: Embeddings OpenAI - Insert
- **Type / role:** OpenAI embeddings provider.
- **Connections:** `ai_embedding` into `Lead Knowledge Base - Insert`.
- **Credentials:** OpenAI.
- **Version notes:** typeVersion 1.2.

#### Node: Document Loader - Lead Data
- **Type / role:** Document loader — converts input to documents for embedding.
- **Config:** Uses default data loader; `textSplittingMode: custom`.
- **Connections:** Receives `ai_textSplitter` from `Text Splitter - Recursive`; outputs `ai_document` to vector insert node.
- **Version notes:** typeVersion 1.1.
- **Failure modes:** Incorrect text extraction; missing fields.

#### Node: Text Splitter - Recursive
- **Type / role:** Text splitter for chunking.
- **Connections:** Feeds `Document Loader - Lead Data` via `ai_textSplitter`.
- **Version notes:** typeVersion 1.
- **Failure modes:** Chunk sizes not configured (defaults may be suboptimal); unexpected splitting of JSON.

#### Node: Store Lead in Database
- **Type / role:** Postgres — persists lead record.
- **Configuration:**
  - Table name is a placeholder value (node field), not dynamically pulled from `Workflow Configuration`.
  - `mappingMode: autoMapInputData`.
- **Connections:** Outputs to `Loop Over Leads`.
- **Version notes:** typeVersion 2.6.
- **Failure modes:** DB credentials missing; table does not exist; column mismatch with auto-mapped fields; JSON/object columns not supported without proper column types.

#### Node: Loop Over Leads
- **Type / role:** Split In Batches — processes leads in batches of 10.
- **Configuration:** `batchSize: 10`.
- **Connections:**
  - Output 0 → `Aggregate Daily Lead Stats`
  - Output 1 → `Lead Enrichment Agent`
- **Important logic implication:**
  - `Merge All Lead Sources` already sends items to `Lead Enrichment Agent` directly, and also to `Loop Over Leads`, which then sends to `Lead Enrichment Agent` again → potential duplication.
  - The batch loop is also connected *after* DB insert (`Store Lead in Database` → `Loop Over Leads`), creating circular/complex execution paths that can cause repeated processing.
- **Version notes:** typeVersion 3.
- **Failure modes:** Infinite loops or repeated processing depending on item flow; confusion about which dataset is being batched.

#### Node: Aggregate Daily Lead Stats
- **Type / role:** Aggregate — aggregates all item data.
- **Configuration:** `aggregateAllItemData`.
- **Connections:** Receives from `Loop Over Leads` output 0.
- **Version notes:** typeVersion 1.
- **Failure modes:** Memory usage on large datasets; unclear output usage (no downstream nodes).

---

### Block 9 — Sticky Notes (Documentation Nodes)
**Overview:** Contains on-canvas notes. In this workflow JSON, these notes describe **e-commerce order processing** and appear unrelated to the real-estate lead routing logic.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`, `Sticky Note5`

- **Important:** n8n JSON does not reliably encode which nodes are visually covered by a sticky note; without the canvas, it’s not possible to duplicate the note content onto all affected nodes with certainty. The summary table below includes sticky note content on the sticky note rows themselves.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point | — | Workflow Configuration |  |
| Workflow Configuration | n8n-nodes-base.set | Central config/constants | Schedule Trigger | Fetch Leads from MLS/Portals; Fetch Leads from CRM/Email/Social |  |
| Fetch Leads from MLS/Portals | n8n-nodes-base.httpRequest | Pull leads from MLS/portals | Workflow Configuration | Merge All Lead Sources |  |
| Fetch Leads from CRM/Email/Social | n8n-nodes-base.httpRequest | Pull leads from CRM/email/social | Workflow Configuration | Merge All Lead Sources |  |
| Merge All Lead Sources | n8n-nodes-base.merge | Combine sources | Fetch Leads from MLS/Portals; Fetch Leads from CRM/Email/Social | Lead Enrichment Agent; Loop Over Leads |  |
| Lead Enrichment Agent | @n8n/n8n-nodes-langchain.agent | LLM-based enrichment | Merge All Lead Sources; Loop Over Leads | Lead Scoring Agent; Lead Sentiment Analysis |  |
| OpenAI Model - Enrichment | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend (enrichment) | — | Lead Enrichment Agent (ai_languageModel) |  |
| Structured Output - Enriched Lead Data | @n8n/n8n-nodes-langchain.outputParserStructured | Enrichment schema enforcement | — | Lead Enrichment Agent (ai_outputParser) |  |
| Lead Sentiment Analysis | @n8n/n8n-nodes-langchain.sentimentAnalysis | Sentiment classification | Lead Enrichment Agent | Check Sentiment Score |  |
| OpenAI Model - Sentiment | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend (sentiment) | — | Lead Sentiment Analysis (ai_languageModel) |  |
| Check Sentiment Score | n8n-nodes-base.if | Gate on Negative/Urgent | Lead Sentiment Analysis | Lead Scoring Agent (true branch only) |  |
| Lead Scoring Agent | @n8n/n8n-nodes-langchain.agent | LLM scoring | Lead Enrichment Agent; Check Sentiment Score | Agent Routing Agent |  |
| OpenAI Model - Scoring | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend (scoring) | — | Lead Scoring Agent (ai_languageModel) |  |
| Structured Output - Lead Score | @n8n/n8n-nodes-langchain.outputParserStructured | Scoring schema enforcement | — | Lead Scoring Agent (ai_outputParser) |  |
| Agent Routing Agent | @n8n/n8n-nodes-langchain.agent | LLM routing w/ tools | Lead Scoring Agent | High-Priority Lead Filter; Fetch Agent Performance Data |  |
| OpenAI Model - Routing | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend (routing) | — | Agent Routing Agent (ai_languageModel) |  |
| Structured Output - Routing Decision | @n8n/n8n-nodes-langchain.outputParserStructured | Routing schema enforcement | — | Agent Routing Agent (ai_outputParser) |  |
| Lead Knowledge Base - Retrieve | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Retrieval tool for routing | — | Agent Routing Agent (ai_tool) |  |
| Embeddings OpenAI - Retrieve | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embeddings for retrieval | — | Lead Knowledge Base - Retrieve (ai_embedding) |  |
| HTTP Request Tool - Property Data | n8n-nodes-base.httpRequestTool | Tool to fetch property data | — | Agent Routing Agent (ai_tool) |  |
| Fetch Agent Performance Data | n8n-nodes-base.httpRequest | Get agent KPIs | Agent Routing Agent | Merge Lead with Agent Data |  |
| Merge Lead with Agent Data | n8n-nodes-base.set | Attach agent performance payload | Fetch Agent Performance Data | High-Priority Lead Filter |  |
| High-Priority Lead Filter | n8n-nodes-base.if | Threshold split | Agent Routing Agent; Merge Lead with Agent Data | Prepare High-Priority Lead Data; Prepare Standard Lead Data |  |
| Prepare High-Priority Lead Data | n8n-nodes-base.set | Add high-priority metadata | High-Priority Lead Filter (true) | Track Engagement Metrics; Notify Slack - High Priority; Send Email - Agent Assignment |  |
| Notify Slack - High Priority | n8n-nodes-base.slack | Slack alert | Prepare High-Priority Lead Data | Track Engagement Metrics |  |
| Send Email - Agent Assignment | n8n-nodes-base.emailSend | Email assigned agent | Prepare High-Priority Lead Data | Track Engagement Metrics |  |
| Prepare Standard Lead Data | n8n-nodes-base.set | Add standard metadata | High-Priority Lead Filter (false) | Track Engagement Metrics |  |
| Track Engagement Metrics | n8n-nodes-base.set | Operational tracking fields | Prepare High-Priority Lead Data; Notify Slack - High Priority; Send Email - Agent Assignment; Prepare Standard Lead Data | Lead Knowledge Base - Insert |  |
| Text Splitter - Recursive | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Chunking for documents | — | Document Loader - Lead Data (ai_textSplitter) |  |
| Document Loader - Lead Data | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Convert item to documents | — | Lead Knowledge Base - Insert (ai_document) |  |
| Embeddings OpenAI - Insert | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embeddings for insert | — | Lead Knowledge Base - Insert (ai_embedding) |  |
| Lead Knowledge Base - Insert | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Store lead vectors | Track Engagement Metrics | Store Lead in Database |  |
| Store Lead in Database | n8n-nodes-base.postgres | Persist to Postgres | Lead Knowledge Base - Insert | Loop Over Leads |  |
| Loop Over Leads | n8n-nodes-base.splitInBatches | Batch looping | Merge All Lead Sources; Store Lead in Database | Aggregate Daily Lead Stats; Lead Enrichment Agent |  |
| Aggregate Daily Lead Stats | n8n-nodes-base.aggregate | Daily aggregation | Loop Over Leads | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas note | — | — | ## Prerequisites / Use Cases / Customization / Benefits (e-commerce oriented) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note | — | — | ## Setup Steps (e-commerce oriented) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note | — | — | ## How It Works (e-commerce oriented) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note | — | — | ## Fulfillment Orchestration and Status Communication (e-commerce oriented) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note | — | — | ## Fraud Detection and Payment Processing (e-commerce oriented) |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas note | — | — | ## Order Intake and Multi-Stage AI Validation (e-commerce oriented) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n. Set it to inactive until credentials and placeholders are configured.

2. **Add “Schedule Trigger”**
   - Node type: *Schedule Trigger*
   - Configure: run daily at **09:00** (or your desired hour/timezone).

3. **Add “Workflow Configuration” (Set node)**
   - Node type: *Set*
   - Enable “Include Other Fields”.
   - Add fields:
     - `mlsApiUrl` (string)
     - `crmApiUrl` (string)
     - `propertyApiUrl` (string)
     - `agentApiUrl` (string)
     - `highPriorityThreshold` (number, e.g., 75)
     - `databaseTable` (string)
     - `slackChannel` (string / channel ID)
     - `senderEmail` (string)
     - (Optional) `enrichmentApiKey` (string; only useful if you later wire it into requests/tools)
   - Connect: **Schedule Trigger → Workflow Configuration**

4. **Add “Fetch Leads from MLS/Portals” (HTTP Request)**
   - Node type: *HTTP Request*
   - URL expression: `{{ $('Workflow Configuration').first().json.mlsApiUrl }}`
   - Response: JSON.
   - Authentication: Header auth (Bearer token) using either:
     - n8n credentials (preferred), or
     - manual header `Authorization: Bearer ...`
   - Connect: **Workflow Configuration → Fetch Leads from MLS/Portals**

5. **Add “Fetch Leads from CRM/Email/Social” (HTTP Request)**
   - Same as above, URL: `{{ $('Workflow Configuration').first().json.crmApiUrl }}`
   - Connect: **Workflow Configuration → Fetch Leads from CRM/Email/Social**

6. **Add “Merge All Lead Sources”**
   - Node type: *Merge*
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - **MLS HTTP → Merge (Input 1)**
     - **CRM HTTP → Merge (Input 2)**

7. **Add “OpenAI Model - Enrichment”**
   - Node type: *OpenAI Chat Model (LangChain)*
   - Model: `gpt-4.1-mini`
   - Credentials: configure **OpenAI API** credential in n8n and select it.

8. **Add “Structured Output - Enriched Lead Data”**
   - Node type: *Structured Output Parser*
   - Schema: create the manual JSON schema for enriched lead data (leadId/name/email/phone + demographics/socialPresence/behavioralData).

9. **Add “Lead Enrichment Agent”**
   - Node type: *AI Agent (LangChain)*
   - Prompt type: define
   - Text: `{{ $json }}`
   - System message: enrichment instructions (demographics, behavior, social signals).
   - Attach:
     - `OpenAI Model - Enrichment` to the agent’s **Language Model** input.
     - `Structured Output - Enriched Lead Data` to the agent’s **Output Parser** input.
   - Connect: **Merge All Lead Sources → Lead Enrichment Agent**

10. **Add “OpenAI Model - Sentiment”**
    - Node type: *OpenAI Chat Model (LangChain)*
    - Model: `gpt-4.1-mini`
    - Use same OpenAI credential.

11. **Add “Lead Sentiment Analysis”**
    - Node type: *Sentiment Analysis (LangChain)*
    - Categories: `Positive, Neutral, Negative, Urgent`
    - Input text: `{{ $json.email }} {{ $json.phone }} {{ $json.name }}`
    - Attach `OpenAI Model - Sentiment` to its language model connector.
    - Connect: **Lead Enrichment Agent → Lead Sentiment Analysis**

12. **Add “Check Sentiment Score” (IF)**
    - Node type: *IF*
    - Condition (OR):
      - `sentimentCategory == "Negative"`
      - `sentimentCategory == "Urgent"`
    - Use expression referencing sentiment output (e.g., `{{ $json.sentimentCategory }}` if the sentiment node outputs it directly).
    - Connect: **Lead Sentiment Analysis → Check Sentiment Score**
    - (Optional but recommended) Decide what to do with the **False** branch (e.g., continue anyway, or route to a different process). The provided workflow only connects the True branch.

13. **Add “OpenAI Model - Scoring”**
    - Node type: OpenAI Chat Model
    - Model: `gpt-4.1-mini`
    - Credentials: OpenAI.

14. **Add “Structured Output - Lead Score”**
    - Node type: Structured Output Parser
    - Schema: leadId, conversionScore (number), priorityLevel enum, indicators array, scoringFactors.

15. **Add “Lead Scoring Agent”**
    - Node type: AI Agent
    - Text: `{{ $json }}`
    - System message: scoring rubric and required output.
    - Attach OpenAI model + score output parser.
    - Connect:
      - **Lead Enrichment Agent → Lead Scoring Agent**
      - (If you want gating) **Check Sentiment Score (true) → Lead Scoring Agent**

16. **Add vector retrieval tool chain for routing**
    - Add **Embeddings OpenAI - Retrieve** (OpenAI embeddings; credential required).
    - Add **Lead Knowledge Base - Retrieve**:
      - Node type: Vector Store In-Memory
      - Mode: `retrieve-as-tool`
      - Memory key: `lead_history`
    - Connect **Embeddings OpenAI - Retrieve → Lead Knowledge Base - Retrieve** (embedding connector).

17. **Add “HTTP Request Tool - Property Data”**
    - Node type: HTTP Request Tool
    - URL: set to your property listings endpoint (preferably use an expression from config).
    - Add auth if required.

18. **Add routing model + parser + agent**
    - Add **OpenAI Model - Routing** (`gpt-4.1-mini`, OpenAI credential).
    - Add **Structured Output - Routing Decision** (schema for assigned agent + rationale + alternatives).
    - Add **Agent Routing Agent**
      - Text: `{{ $json }}`
      - System message: routing criteria.
      - Attach:
        - OpenAI routing model
        - routing output parser
        - tools: `Lead Knowledge Base - Retrieve` and `HTTP Request Tool - Property Data`
    - Connect: **Lead Scoring Agent → Agent Routing Agent**

19. **Add “Fetch Agent Performance Data” + “Merge Lead with Agent Data” (optional integration path)**
    - Add HTTP Request node to agent performance API (Bearer auth).
    - Add Set node to attach results as `agentPerformanceData`.
    - Connect: **Agent Routing Agent → Fetch Agent Performance Data → Merge Lead with Agent Data**
    - If you truly need to combine routing decision + performance response, use a proper **Merge (by key)** or restructure so performance is fetched *before* routing and included in routing input.

20. **Add “High-Priority Lead Filter”**
    - Node type: IF
    - Condition: `{{ $json.conversionScore }} >= {{ $('Workflow Configuration').first().json.highPriorityThreshold }}`
    - Connect incoming: **Agent Routing Agent → High-Priority Lead Filter**
    - (If you keep the agent performance branch) connect **Merge Lead with Agent Data → High-Priority Lead Filter** only if the payload contains `conversionScore` and routing output; otherwise merge properly first.

21. **Add “Prepare High-Priority Lead Data” (Set)**
    - Add fields: `priority`, `notificationUrgency`, `followUpWindow`, `assignmentTimestamp: {{ $now.toISO() }}`
    - Connect: **High-Priority Lead Filter (true) → Prepare High-Priority Lead Data**

22. **Add Slack notification**
    - Node type: Slack
    - Auth: configure **Slack OAuth2** credential (scopes to post messages).
    - Channel: set channel ID (ideally from config).
    - Message template uses fields like `name`, `conversionScore`, `assignedAgent.agentName`, `routingRationale`.
    - Connect: **Prepare High-Priority Lead Data → Notify Slack - High Priority**

23. **Add email notification**
    - Node type: Email Send
    - Configure SMTP (or your n8n email provider).
    - From: set to `{{ $('Workflow Configuration').first().json.senderEmail }}` (recommended)
    - To: ensure routing output includes an email field; either:
      - extend routing schema to include `assignedAgent.agentEmail`, or
      - map agentId → email via a lookup table/API before sending.
    - Connect: **Prepare High-Priority Lead Data → Send Email - Agent Assignment**

24. **Add “Prepare Standard Lead Data”**
    - Node type: Set
    - Add standard metadata and timestamp.
    - Connect: **High-Priority Lead Filter (false) → Prepare Standard Lead Data**

25. **Add “Track Engagement Metrics”**
    - Node type: Set
    - Add engagement fields and `roiTrackingId`.
    - Connect both:
      - **Prepare High-Priority Lead Data → Track Engagement Metrics**
      - **Prepare Standard Lead Data → Track Engagement Metrics**
    - (If Slack/email also feed into it, be aware you may create duplicate items—prefer a single path.)

26. **Add vector insert chain**
    - Add **Text Splitter - Recursive**
    - Add **Document Loader - Lead Data** (custom split mode)
    - Add **Embeddings OpenAI - Insert**
    - Add **Lead Knowledge Base - Insert**:
      - Mode: insert
      - Memory key: `lead_history`
    - Connect:
      - **Text Splitter → Document Loader** (text splitter connector)
      - **Document Loader → Lead Knowledge Base - Insert** (document connector)
      - **Embeddings OpenAI - Insert → Lead Knowledge Base - Insert** (embedding connector)
      - **Track Engagement Metrics → Lead Knowledge Base - Insert** (main)

27. **Add “Store Lead in Database” (Postgres)**
    - Node type: Postgres
    - Credentials: configure Postgres connection.
    - Schema: public
    - Table: set your leads table (create it first with columns matching your stored fields, or use JSONB columns).
    - Mapping: auto-map input.
    - Connect: **Lead Knowledge Base - Insert → Store Lead in Database**

28. **Add batch processing + aggregation (optional)**
    - Add **Split In Batches** (`batchSize: 10`)
    - Add **Aggregate** (aggregate all item data)
    - If you want batching, feed the *original lead list* into SplitInBatches and continue from there; avoid circular connections that re-insert already stored leads.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The sticky note content embedded in this workflow canvas describes an **e-commerce order fulfillment** scenario (webhook intake, fraud screening, payment, shipping), which does not match the real-estate lead routing nodes. | Review/remove or replace these notes to avoid operator confusion. |
| Multiple connections create parallel paths that can duplicate scoring/routing or pass incompatible payloads into the priority filter (e.g., agent performance response lacking `conversionScore`). | Consider restructuring with explicit merges (by key) and a single authoritative lead object per stage. |
| In-memory vector store (`lead_history`) is typically **not persistent** across separate workflow executions. | Use a persistent vector DB (or n8n’s supported vector stores) if you need long-term memory. |
| Routing email step references `assignedAgent.agentEmail` but the routing output schema doesn’t define it. | Add `agentEmail` to the routing schema and prompt, or perform a lookup by agentId before emailing. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.