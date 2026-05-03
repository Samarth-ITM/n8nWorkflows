Predict tenant churn and manage retention with GPT-4, Gmail, Slack and your CRM

https://n8nworkflows.xyz/workflows/predict-tenant-churn-and-manage-retention-with-gpt-4--gmail--slack-and-your-crm-12038


# Predict tenant churn and manage retention with GPT-4, Gmail, Slack and your CRM

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Smart AI-Driven Tenant Experience & Retention Manager  
**Title (given):** Predict tenant churn and manage retention with GPT-4, Gmail, Slack and your CRM

**Purpose:**  
This workflow runs on a daily schedule to fetch tenant-related data (tenant profiles, service requests, feedback), consolidate it, use GPT-4 to score satisfaction and churn risk, then automatically triggers retention actions:
- **High churn risk:** GPT-4 crafts personalized retention communications, sends an email via Gmail and alerts the team via Slack.
- **Lower churn risk:** sends a loyalty reward email.
Finally, it updates a CRM and a retention dashboard with the latest analysis.

### 1.1 Scheduled Execution & Configuration
Runs every day at a fixed hour and sets environment/config variables (API endpoints and churn threshold).

### 1.2 Data Retrieval (Tenant / Service Requests / Feedback)
Calls 3 HTTP endpoints to obtain data required for churn analysis.

### 1.3 Consolidation of Tenant Intelligence
Combines retrieved datasets into a single payload for AI analysis.

### 1.4 AI Churn / Satisfaction Analysis (GPT-4 + Structured Parsing)
GPT-4 analyzes the merged data and outputs a structured object (tenantId, scores, risk level, issues, actions).

### 1.5 Decisioning (High risk vs. Low/Medium)
Compares the churn score to a configurable threshold.

### 1.6 Retention Actions & Notifications
- High-risk path: GPT-4 agent uses Gmail Tool + Slack Tool to execute comms.
- Low-risk path: sends a templated loyalty email.

### 1.7 Systems of Record Updates
Posts results to CRM and retention dashboard APIs.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Execution & Configuration

**Overview:**  
Triggers daily and centralizes workflow configuration (API endpoints and churn threshold) in one node so downstream nodes reference it consistently.

**Nodes involved:**
- Daily Tenant Analysis Schedule
- Workflow Configuration

#### Node: Daily Tenant Analysis Schedule
- **Type / role:** Schedule Trigger — entry point; starts workflow automatically.
- **Configuration:** Runs daily with `triggerAtHour: 8` (server time).
- **Connections:**  
  - Output → **Workflow Configuration**
- **Potential failures / edge cases:**
  - Timezone mismatch (n8n instance timezone vs desired business timezone).
  - If execution overlaps with previous run (large datasets), consider concurrency settings.

#### Node: Workflow Configuration
- **Type / role:** Set node — defines configuration variables used across the workflow.
- **Key fields set (placeholders to replace):**
  - `tenantApiUrl` (string)
  - `serviceRequestApiUrl` (string)
  - `feedbackApiUrl` (string)
  - `crmApiUrl` (string)
  - `dashboardApiUrl` (string)
  - `churnRiskThreshold` (number) = **70**
- **Notable setting:** `includeOtherFields: true` (keeps incoming JSON; here mainly irrelevant since this is right after trigger).
- **Connections:**  
  - Output → **Fetch Tenant Data**, **Fetch Service Requests**, **Fetch Feedback Data** (fan-out)
- **Potential failures / edge cases:**
  - If placeholders are not replaced with real URLs, HTTP nodes will fail.
  - Threshold choice drives routing; ensure it matches your risk policy.

**Sticky note coverage (contextual):**
- *How It Works* (high-level description)
- *Setup Steps* (configure sources, OpenAI, Gmail/Slack/CRM, thresholds)
- *Prerequisites* and *Customization/Benefits*

---

### Block 2 — Data Retrieval (Tenant / Service Requests / Feedback)

**Overview:**  
Fetches data from three external systems/APIs. Each request uses the URL defined in the configuration node.

**Nodes involved:**
- Fetch Tenant Data
- Fetch Service Requests
- Fetch Feedback Data

#### Node: Fetch Tenant Data
- **Type / role:** HTTP Request — retrieves tenant profile/engagement dataset.
- **Configuration choices:**
  - URL: `{{ $('Workflow Configuration').first().json.tenantApiUrl }}`
  - Sends header `Content-Type: application/json`
  - Method not explicitly set (defaults to GET in n8n HTTP Request)
- **Connections:**  
  - Output → **Merge Tenant Intelligence**
- **Potential failures / edge cases:**
  - Missing auth (none configured here); real APIs usually require OAuth2/API key.
  - Non-JSON responses or unexpected schema will degrade AI analysis.
  - Pagination not handled: if API returns paginated results, only the first page is processed unless you implement paging.

#### Node: Fetch Service Requests
- **Type / role:** HTTP Request — retrieves maintenance/service ticket history.
- **Configuration:**
  - URL from `serviceRequestApiUrl`
  - Header `Content-Type: application/json`
- **Connections:**  
  - Output → **Merge Tenant Intelligence**
- **Potential failures / edge cases:**
  - Large payloads can increase execution time and token usage downstream.
  - If service requests aren’t linked to tenant IDs in a consistent way, analysis will be unreliable.

#### Node: Fetch Feedback Data
- **Type / role:** HTTP Request — retrieves survey/NPS/feedback dataset.
- **Configuration:**
  - URL from `feedbackApiUrl`
  - Header `Content-Type: application/json`
- **Connections:**  
  - Output → **Merge Tenant Intelligence**
- **Potential failures / edge cases:**
  - Feedback data often contains free text; ensure encoding/escaping is consistent.
  - If API returns nested items arrays, downstream “all()” usage may create a multi-item structure you didn’t intend.

**Sticky note coverage (contextual):**
- *Consolidate Tenant Intelligence* (applies to this retrieval + merge region)

---

### Block 3 — Consolidation of Tenant Intelligence

**Overview:**  
Aggregates the outputs of the three HTTP calls into a single JSON structure that is passed to the AI analysis agent.

**Nodes involved:**
- Merge Tenant Intelligence

#### Node: Merge Tenant Intelligence
- **Type / role:** Set node — constructs a unified payload.
- **Configuration choices (expressions):**
  - `tenantData` (object): `{{ $('Fetch Tenant Data').all() }}`
  - `serviceRequests` (array): `{{ $('Fetch Service Requests').all() }}`
  - `feedbackData` (array): `{{ $('Fetch Feedback Data').all() }}`
- **Important behavior:**  
  `all()` returns **all items** from that node. This means the merged payload can include arrays of items rather than a single normalized structure per tenant.
- **Connections:**  
  - Output → **Tenant Analysis Agent**
- **Potential failures / edge cases:**
  - **Misalignment risk:** The workflow does not explicitly map service requests/feedback to a specific tenant item. If the APIs return multiple tenants, the AI may receive a “portfolio dump” rather than per-tenant bundles.
  - Memory/token explosion if datasets are large (`JSON.stringify($json)` later).
  - If one upstream HTTP call fails or returns zero items, merged fields may be empty arrays.

**Sticky note coverage (contextual):**
- *Consolidate Tenant Intelligence* (What/Why)

---

### Block 4 — AI Churn / Satisfaction Analysis (GPT-4 + Structured Parsing)

**Overview:**  
Uses a LangChain Agent backed by GPT-4o to analyze the merged dataset and output a structured churn/satisfaction assessment.

**Nodes involved:**
- Tenant Analysis Agent
- OpenAI GPT-4
- Analysis Output Parser

#### Node: OpenAI GPT-4
- **Type / role:** LangChain Chat Model — provides the LLM for the analysis agent.
- **Model:** `gpt-4o`
- **Credentials:** `openAiApi` (must be configured in n8n)
- **Connections:**  
  - ai_languageModel → **Tenant Analysis Agent**
- **Potential failures / edge cases:**
  - Invalid API key / quota exceeded.
  - Model availability or policy changes.
  - Latency/timeouts with large prompts.

#### Node: Analysis Output Parser
- **Type / role:** Structured Output Parser — enforces a JSON schema for the agent output.
- **Schema (manual) key fields:**
  - `tenantId` (string)
  - `tenantName` (string)
  - `satisfactionScore` (number 0–100 expected)
  - `churnRiskScore` (number 0–100 expected)
  - `churnRiskLevel` (“low” | “medium” | “high” expected)
  - `keyIssues` (array of strings)
  - `recommendedActions` (array of strings)
  - `engagementLevel` (“low” | “medium” | “high” expected)
- **Connections:**  
  - ai_outputParser → **Tenant Analysis Agent**
- **Potential failures / edge cases:**
  - If the model outputs text not matching schema, parsing can fail or return null/partial.
  - No required fields are explicitly enforced in the shown schema; consider adding `required` for reliability.

#### Node: Tenant Analysis Agent
- **Type / role:** LangChain Agent — runs analysis prompt and returns structured output.
- **Prompt text:**  
  `Analyze this tenant data: {{ JSON.stringify($json) }}`
- **System message:** instructs the agent to:
  - analyze engagement/service/feedback
  - compute satisfaction and churn risk scores
  - classify churn risk level
  - identify issues and propose retention actions
  - assess engagement level
- **Connections:**  
  - Main output → **Check Churn Risk Level**
  - Receives model via **OpenAI GPT-4**
  - Uses **Analysis Output Parser**
- **Potential failures / edge cases:**
  - Prompt may exceed context window if `$json` contains large arrays.
  - If data is not tenant-specific (portfolio-level), output may be for “one tenant” only, ignoring others.
  - Scores may be non-numeric if model misbehaves; downstream IF expects numeric.

**Sticky note coverage (contextual):**
- *Assess Churn Risk* (What/Why)

---

### Block 5 — Decisioning (Threshold Check)

**Overview:**  
Routes execution based on whether churn risk score meets/exceeds the configured threshold.

**Nodes involved:**
- Check Churn Risk Level

#### Node: Check Churn Risk Level
- **Type / role:** IF node — branching logic.
- **Condition:**  
  `Tenant Analysis Agent.item.json.churnRiskScore >= Workflow Configuration.first().json.churnRiskThreshold`
  - Default threshold: **70**
- **Connections (two branches):**
  - **True** → **High Risk Communication Agent**
  - **False** → **Low Risk Engagement**
- **Potential failures / edge cases:**
  - If `churnRiskScore` is missing/NaN/string, “loose” validation may behave unexpectedly.
  - If you intend “high risk” to mean `churnRiskLevel == "high"` you must align threshold logic with the model’s classification.

---

### Block 6 — Retention Actions & Notifications

**Overview:**  
Executes either high-risk personalized interventions (AI-generated email + Slack alert) or low-risk loyalty outreach.

**Nodes involved:**
- High Risk Communication Agent
- OpenAI GPT-4 for Communications
- Gmail Tool
- Slack Tool
- Low Risk Engagement
- Send Loyalty Reward Email

#### Node: OpenAI GPT-4 for Communications
- **Type / role:** LangChain Chat Model — provides the LLM for the communications agent.
- **Model:** `gpt-4o`
- **Credentials:** `openAiApi`
- **Connections:**  
  - ai_languageModel → **High Risk Communication Agent**
- **Potential failures:** same as other OpenAI node.

#### Node: High Risk Communication Agent
- **Type / role:** LangChain Agent — orchestrates comms using AI tools.
- **Input text (context):** includes tenant name/id, churn score, key issues, recommended actions.
- **System message:** instructs agent to:
  1) use **Gmail Tool** to send a personalized retention email  
  2) use **Slack Tool** to notify the internal team  
  and confirm completion
- **Tool connections:**
  - ai_tool → Gmail Tool
  - ai_tool → Slack Tool
- **Main output:** → **Update CRM**
- **Potential failures / edge cases:**
  - If AI does not call tools, no email/Slack message will be sent.
  - Missing tenant email address: Gmail Tool expects `recipientEmail` via `$fromAI(...)`.
  - Compliance: ensure incentives/offers are valid and approved.
  - If Slack channel ID is wrong, Slack Tool fails.

#### Node: Gmail Tool
- **Type / role:** Gmail Tool (AI Tool) — sends email using parameters generated by the AI agent.
- **Key fields (AI-provided via `$fromAI`):**
  - `sendTo`: `recipientEmail`
  - `subject`: `emailSubject`
  - `message` (HTML): `emailBody`
- **Credentials:** Gmail OAuth2
- **Connections:** Used by **High Risk Communication Agent** as an AI tool.
- **Potential failures / edge cases:**
  - OAuth token expired / missing scopes (gmail.send).
  - AI might produce non-HTML or unsafe HTML; consider sanitization policies.
  - Invalid recipient email format.

#### Node: Slack Tool
- **Type / role:** Slack Tool (AI Tool) — posts a message to a channel chosen/provided by AI.
- **Key fields (AI-provided via `$fromAI`):**
  - `text`: `slackMessage`
  - `channelId`: `slackChannel`
  - Authentication: OAuth2
- **Credentials:** Slack OAuth2
- **Connections:** Used by **High Risk Communication Agent** as an AI tool.
- **Potential failures / edge cases:**
  - Bot/user not in channel, missing permission (chat:write).
  - AI chooses an incorrect channel ID; safer to hardcode a channel in node config.

#### Node: Low Risk Engagement
- **Type / role:** Set node — creates a default loyalty email template.
- **Fields set:**
  - `emailSubject`: “Thank You for Being a Valued Tenant!”
  - `emailBody`: HTML template referencing `{{ $json.tenantName }}`
- **Connections:**  
  - Output → **Send Loyalty Reward Email**
- **Potential failures / edge cases:**
  - Requires `tenantName` in incoming JSON; otherwise template will render blank.
  - Message promises a reward; ensure your business process can fulfill it.

#### Node: Send Loyalty Reward Email
- **Type / role:** Gmail node (standard, not AI Tool) — sends email for low-risk tenants.
- **Key configuration:**
  - To: `{{ $json.tenantEmail || '<__PLACEHOLDER_VALUE__Tenant email field from data__>' }}`
  - Subject: `{{ $json.emailSubject }}`
  - Body: `{{ $json.emailBody }}`
- **Credentials:** Gmail OAuth2
- **Connections:**  
  - Output → **Update CRM**
- **Potential failures / edge cases:**
  - `tenantEmail` may not exist in AI output; the fallback placeholder must be replaced with a real field mapping.
  - If multiple tenants are intended, this workflow currently sends only for whichever item reaches this branch.

**Sticky note coverage (contextual):**
- *Execute Retention Actions and Updates* (communications and campaigns)

---

### Block 7 — Systems of Record Updates (CRM + Dashboard)

**Overview:**  
Posts analysis results and actions taken to a CRM and a dashboard endpoint for tracking and reporting.

**Nodes involved:**
- Update CRM
- Update Retention Dashboard

#### Node: Update CRM
- **Type / role:** HTTP Request — writes analysis record to CRM.
- **Configuration:**
  - Method: POST
  - URL: `{{ $('Workflow Configuration').first().json.crmApiUrl }}`
  - JSON Body includes:
    - `tenantId`, `satisfactionScore`, `churnRiskScore`, `churnRiskLevel`
    - `lastAnalyzed`: `new Date().toISOString()`
    - `actionsTaken`: `$json.recommendedActions`
  - Header: `Content-Type: application/json`
- **Connections:**  
  - Output → **Update Retention Dashboard**
- **Potential failures / edge cases:**
  - `recommendedActions` may be undefined on low-risk path (since Low Risk Engagement doesn’t set it). CRM payload may contain null/undefined.
  - Auth not configured (most CRMs require auth).
  - Rate limits and idempotency: repeated daily runs should upsert rather than duplicate; current config always POSTs.

#### Node: Update Retention Dashboard
- **Type / role:** HTTP Request — updates dashboard store for reporting/visualization.
- **Configuration:**
  - Method: POST
  - URL: `{{ $('Workflow Configuration').first().json.dashboardApiUrl }}`
  - JSON Body includes:
    - `tenantId`, `tenantName`
    - `satisfactionScore`, `churnRiskScore`, `churnRiskLevel`
    - `engagementLevel`
    - `timestamp`: `new Date().toISOString()`
- **Connections:** none (end)
- **Potential failures / edge cases:**
  - Missing `tenantName` / `engagementLevel` if upstream output is incomplete.
  - Needs auth and error handling; not present.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Tenant Analysis Schedule | scheduleTrigger | Daily entry point | — | Workflow Configuration | ## How It Works<br>Automates daily tenant analytics, churn risk evaluation, and proactive retention by unifying tenant data from multiple sources, applying GPT-4–based risk scoring, detecting early churn indicators, routing high-risk tenants to retention specialists, and initiating targeted engagement campaigns. It retrieves tenant profiles, service requests, and feedback data, performs GPT-4 analysis with detailed churn risk insights, assesses engagement levels, escalates high-risk tenants to dedicated communication teams, delivers personalized loyalty incentives and engagement emails, and updates CRM systems and retention dashboards. Designed for property management companies and SaaS providers. |
| Workflow Configuration | set | Central config variables | Daily Tenant Analysis Schedule | Fetch Tenant Data; Fetch Service Requests; Fetch Feedback Data | ## Setup Steps<br>1. Configure tenant data sources.<br>2. Connect OpenAI GPT-4 API for risk analysis and churn prediction.<br>3. Set up Gmail, Slack, and CRM credentials for communication and tracking.<br>4. Define churn risk thresholds, retention messaging templates, and reward programs. |
| Fetch Tenant Data | httpRequest | Retrieve tenant dataset | Workflow Configuration | Merge Tenant Intelligence | ## Consolidate Tenant Intelligence<br>What: Merges tenant data from service requests, feedback, and engagement systems.<br>Why: Provides holistic view of tenant health and satisfaction across touchpoints. |
| Fetch Service Requests | httpRequest | Retrieve service request history | Workflow Configuration | Merge Tenant Intelligence | ## Consolidate Tenant Intelligence<br>What: Merges tenant data from service requests, feedback, and engagement systems.<br>Why: Provides holistic view of tenant health and satisfaction across touchpoints. |
| Fetch Feedback Data | httpRequest | Retrieve feedback dataset | Workflow Configuration | Merge Tenant Intelligence | ## Consolidate Tenant Intelligence<br>What: Merges tenant data from service requests, feedback, and engagement systems.<br>Why: Provides holistic view of tenant health and satisfaction across touchpoints. |
| Merge Tenant Intelligence | set | Build unified AI input payload | Fetch Tenant Data; Fetch Service Requests; Fetch Feedback Data | Tenant Analysis Agent | ## Consolidate Tenant Intelligence<br>What: Merges tenant data from service requests, feedback, and engagement systems.<br>Why: Provides holistic view of tenant health and satisfaction across touchpoints. |
| Tenant Analysis Agent | langchain.agent | GPT-based analysis + scoring | Merge Tenant Intelligence; OpenAI GPT-4; Analysis Output Parser | Check Churn Risk Level | ## Assess Churn Risk<br>What: Uses GPT-4 to analyze behavioral signals and assign churn risk scores.<br>Why: Identifies at-risk tenants early for proactive intervention. |
| OpenAI GPT-4 | lmChatOpenAi | LLM for analysis | — | Tenant Analysis Agent (ai_languageModel) | ## Assess Churn Risk<br>What: Uses GPT-4 to analyze behavioral signals and assign churn risk scores.<br>Why: Identifies at-risk tenants early for proactive intervention. |
| Analysis Output Parser | outputParserStructured | Enforce structured AI output | — | Tenant Analysis Agent (ai_outputParser) | ## Assess Churn Risk<br>What: Uses GPT-4 to analyze behavioral signals and assign churn risk scores.<br>Why: Identifies at-risk tenants early for proactive intervention. |
| Check Churn Risk Level | if | Route high-risk vs low-risk actions | Tenant Analysis Agent | High Risk Communication Agent; Low Risk Engagement | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| High Risk Communication Agent | langchain.agent | Orchestrate Gmail+Slack comms for high-risk tenants | Check Churn Risk Level; OpenAI GPT-4 for Communications; Gmail Tool; Slack Tool | Update CRM | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| OpenAI GPT-4 for Communications | lmChatOpenAi | LLM for comms agent | — | High Risk Communication Agent (ai_languageModel) | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| Gmail Tool | gmailTool | AI tool: send personalized retention email | High Risk Communication Agent (ai_tool) | — | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| Slack Tool | slackTool | AI tool: notify internal team | High Risk Communication Agent (ai_tool) | — | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| Low Risk Engagement | set | Create loyalty email template | Check Churn Risk Level | Send Loyalty Reward Email | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| Send Loyalty Reward Email | gmail | Send templated loyalty email | Low Risk Engagement | Update CRM | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| Update CRM | httpRequest | Persist scores/actions to CRM | High Risk Communication Agent; Send Loyalty Reward Email | Update Retention Dashboard | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| Update Retention Dashboard | httpRequest | Post metrics to dashboard | Update CRM | — | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| Sticky Note | stickyNote | Comment | — | — | ## Customization<br>Adjust churn risk algorithms and thresholds,<br><br>## Benefits<br>Predicts churn before it happens, enables proactive retention |
| Sticky Note1 | stickyNote | Comment | — | — | ## Prerequisites<br>Tenant/customer data source; service request system; feedback collection tool;<br><br>## Use Cases<br>Property management automating tenant retention across portfolios; |
| Sticky Note2 | stickyNote | Comment | — | — | ## Setup Steps<br>1. Configure tenant data sources.<br>2. Connect OpenAI GPT-4 API for risk analysis and churn prediction.<br>3. Set up Gmail, Slack, and CRM credentials for communication and tracking.<br>4. Define churn risk thresholds, retention messaging templates, and reward programs. |
| Sticky Note3 | stickyNote | Comment | — | — | ## How It Works<br>Automates daily tenant analytics, churn risk evaluation, and proactive retention by unifying tenant data from multiple sources, applying GPT-4–based risk scoring, detecting early churn indicators, routing high-risk tenants to retention specialists, and initiating targeted engagement campaigns. It retrieves tenant profiles, service requests, and feedback data, performs GPT-4 analysis with detailed churn risk insights, assesses engagement levels, escalates high-risk tenants to dedicated communication teams, delivers personalized loyalty incentives and engagement emails, and updates CRM systems and retention dashboards. Designed for property management companies and SaaS providers. |
| Sticky Note4 | stickyNote | Comment | — | — | ## Execute Retention Actions and Updates<br>What: Sends personalized communications, loyalty rewards, and engagement campaigns .<br>Why: Delivers timely, relevant interventions that reduce churn and strengthen relationships. |
| Sticky Note5 | stickyNote | Comment | — | — | ## Assess Churn Risk<br>What: Uses GPT-4 to analyze behavioral signals and assign churn risk scores.<br>Why: Identifies at-risk tenants early for proactive intervention. |
| Sticky Note6 | stickyNote | Comment | — | — | ## Consolidate Tenant Intelligence<br>What: Merges tenant data from service requests, feedback, and engagement systems.<br>Why: Provides holistic view of tenant health and satisfaction across touchpoints. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Smart AI-Driven Tenant Experience & Retention Manager**
- Keep it inactive until credentials and endpoints are set.

2) **Add trigger: “Schedule Trigger”**
- Node name: **Daily Tenant Analysis Schedule**
- Configure: Daily at **08:00** (adjust timezone as needed).

3) **Add configuration node: “Set”**
- Node name: **Workflow Configuration**
- Add fields:
  - `tenantApiUrl` (string) → your tenant API endpoint
  - `serviceRequestApiUrl` (string) → your service request/ticket endpoint
  - `feedbackApiUrl` (string) → your feedback/survey endpoint
  - `crmApiUrl` (string) → your CRM ingestion endpoint
  - `dashboardApiUrl` (string) → your dashboard ingestion endpoint
  - `churnRiskThreshold` (number) → **70** (or your chosen threshold)
- Connect: **Daily Tenant Analysis Schedule → Workflow Configuration**

4) **Add three “HTTP Request” nodes (GET)**
- **Fetch Tenant Data**
  - URL: `{{$('Workflow Configuration').first().json.tenantApiUrl}}`
  - Header: `Content-Type: application/json`
- **Fetch Service Requests**
  - URL: `{{$('Workflow Configuration').first().json.serviceRequestApiUrl}}`
  - Header: `Content-Type: application/json`
- **Fetch Feedback Data**
  - URL: `{{$('Workflow Configuration').first().json.feedbackApiUrl}}`
  - Header: `Content-Type: application/json`
- Connect: **Workflow Configuration →** each of the three HTTP nodes (3 outgoing connections).
- If your APIs need authentication, configure it in each HTTP node (API key/OAuth2/etc.).

5) **Add “Set” node to consolidate**
- Node name: **Merge Tenant Intelligence**
- Create fields:
  - `tenantData` (object) = `{{$('Fetch Tenant Data').all()}}`
  - `serviceRequests` (array) = `{{$('Fetch Service Requests').all()}}`
  - `feedbackData` (array) = `{{$('Fetch Feedback Data').all()}}`
- Connect: each HTTP node → **Merge Tenant Intelligence**

6) **Add AI components for analysis**
- Add **OpenAI Chat Model** node (LangChain):
  - Name: **OpenAI GPT-4**
  - Model: **gpt-4o**
  - Set OpenAI credential (API key) in n8n Credentials.
- Add **Structured Output Parser** node:
  - Name: **Analysis Output Parser**
  - Schema: create the object schema with fields: tenantId, tenantName, satisfactionScore, churnRiskScore, churnRiskLevel, keyIssues[], recommendedActions[], engagementLevel.
- Add **AI Agent** node:
  - Name: **Tenant Analysis Agent**
  - Prompt: `Analyze this tenant data: {{ JSON.stringify($json) }}`
  - System message: tenant churn/satisfaction instructions (as in the workflow)
- Connect:
  - **Merge Tenant Intelligence → Tenant Analysis Agent** (main)
  - **OpenAI GPT-4 → Tenant Analysis Agent** (AI language model connection)
  - **Analysis Output Parser → Tenant Analysis Agent** (AI output parser connection)

7) **Add IF decision node**
- Node name: **Check Churn Risk Level**
- Condition (Number, >=):
  - Left: `{{$('Tenant Analysis Agent').item.json.churnRiskScore}}`
  - Right: `{{$('Workflow Configuration').first().json.churnRiskThreshold}}`
- Connect: **Tenant Analysis Agent → Check Churn Risk Level**

8) **High-risk branch: add communications agent + tools**
- Add **OpenAI Chat Model** node:
  - Name: **OpenAI GPT-4 for Communications**
  - Model: **gpt-4o**
  - Credential: same OpenAI credential
- Add **AI Agent** node:
  - Name: **High Risk Communication Agent**
  - Text includes tenantName/tenantId/churn score/keyIssues/recommendedActions
  - System message instructs sending Gmail + Slack and confirming completion
- Add **Gmail Tool** node:
  - Name: **Gmail Tool**
  - Configure Gmail OAuth2 credential in n8n Credentials (scopes to send email).
  - Map fields using `$fromAI`:
    - To: `recipientEmail`
    - Subject: `emailSubject`
    - Message HTML: `emailBody`
- Add **Slack Tool** node:
  - Name: **Slack Tool**
  - Configure Slack OAuth2 credential (chat:write).
  - Map:
    - ChannelId: `$fromAI('slackChannel', ...)` (or hardcode a known channel ID for safety)
    - Text: `$fromAI('slackMessage', ...)`
- Connect:
  - IF **true** → **High Risk Communication Agent**
  - **OpenAI GPT-4 for Communications → High Risk Communication Agent** (AI model)
  - **Gmail Tool → High Risk Communication Agent** (AI tool connection)
  - **Slack Tool → High Risk Communication Agent** (AI tool connection)

9) **Low-risk branch: add templated email**
- Add **Set** node:
  - Name: **Low Risk Engagement**
  - Fields:
    - `emailSubject` = “Thank You for Being a Valued Tenant!”
    - `emailBody` = your HTML loyalty template referencing `{{$json.tenantName}}`
- Add **Gmail** node (standard):
  - Name: **Send Loyalty Reward Email**
  - To: `{{$json.tenantEmail}}` (map correctly from your data; avoid placeholders)
  - Subject: `{{$json.emailSubject}}`
  - Message: `{{$json.emailBody}}`
  - Credential: Gmail OAuth2
- Connect:
  - IF **false** → **Low Risk Engagement → Send Loyalty Reward Email**

10) **Add CRM update (HTTP Request POST)**
- Node name: **Update CRM**
- Method: POST
- URL: `{{$('Workflow Configuration').first().json.crmApiUrl}}`
- JSON body (example structure):
  - tenantId, satisfactionScore, churnRiskScore, churnRiskLevel
  - lastAnalyzed = `new Date().toISOString()`
  - actionsTaken = `{{$json.recommendedActions}}`
- Header: `Content-Type: application/json`
- Connect:
  - **High Risk Communication Agent → Update CRM**
  - **Send Loyalty Reward Email → Update CRM**

11) **Add dashboard update (HTTP Request POST)**
- Node name: **Update Retention Dashboard**
- Method: POST
- URL: `{{$('Workflow Configuration').first().json.dashboardApiUrl}}`
- JSON body:
  - tenantId, tenantName, satisfactionScore, churnRiskScore, churnRiskLevel, engagementLevel
  - timestamp = `new Date().toISOString()`
- Header: `Content-Type: application/json`
- Connect: **Update CRM → Update Retention Dashboard**

12) **Validate credentials and placeholders**
- Replace all `"<__PLACEHOLDER_VALUE__...>"` with real values.
- Ensure Gmail/Slack/OpenAI credentials are valid and tested.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Customization: Adjust churn risk algorithms and thresholds. Benefits: Predicts churn before it happens, enables proactive retention. | Sticky note “Customization / Benefits” |
| Prerequisites: Tenant/customer data source; service request system; feedback collection tool. Use cases: property management automating tenant retention across portfolios. | Sticky note “Prerequisites / Use Cases” |
| Setup steps: configure data sources, connect OpenAI, set up Gmail/Slack/CRM credentials, define thresholds and messaging/rewards. | Sticky note “Setup Steps” |
| How it works: daily analytics → GPT-4 scoring → route high-risk to comms → loyalty emails → update CRM/dashboard. | Sticky note “How It Works” |
| Assess churn risk: uses GPT-4 behavioral analysis to assign churn risk scores for proactive intervention. | Sticky note “Assess Churn Risk” |
| Execute retention actions and updates: sends personalized communications and updates systems to reduce churn. | Sticky note “Execute Retention Actions and Updates” |
| Consolidate tenant intelligence: merge service requests, feedback, engagement for a holistic view. | Sticky note “Consolidate Tenant Intelligence” |