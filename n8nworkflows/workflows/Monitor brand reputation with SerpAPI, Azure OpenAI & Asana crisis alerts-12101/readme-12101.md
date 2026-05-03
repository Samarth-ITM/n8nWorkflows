Monitor brand reputation with SerpAPI, Azure OpenAI & Asana crisis alerts

https://n8nworkflows.xyz/workflows/monitor-brand-reputation-with-serpapi--azure-openai---asana-crisis-alerts-12101


# Monitor brand reputation with SerpAPI, Azure OpenAI & Asana crisis alerts

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow monitors online brand mentions (notably Reddit and review sites like Glassdoor/Clutch) on an hourly basis using **SerpAPI Google AI Mode**, analyzes each extracted “insight” with **Azure OpenAI** for sentiment and crisis risk, and triggers **crisis response actions** when risk is classified as **High** (Google Chat alert + Asana task).

**Target use cases**
- Early detection of reputational threats (public complaints, allegations, viral negativity)
- Ongoing brand monitoring across forums and review platforms
- Automated incident escalation into team comms (Google Chat) and task management (Asana)

### Logical blocks
1. **1.1 Scheduled Trigger** → start workflow every hour  
2. **1.2 Search & Data Collection (SerpAPI)** → run Google AI Mode query to gather insights + references  
3. **1.3 Parsing & Normalization** → convert SerpAPI response into multiple structured n8n items (one per insight/reference)  
4. **1.4 AI Risk Detection (Azure OpenAI via LangChain Agent)** → classify sentiment/risk; enforce strict JSON output  
5. **1.5 Post-processing & Filtering** → parse/clean AI output; keep only High risk  
6. **1.6 Crisis Response** → Google Chat alert and Asana urgent task

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled Trigger

**Overview:** Kicks off the monitoring run automatically every hour.

**Nodes involved**
- **Schedule: Every Hour**

**Node details**
- **Schedule: Every Hour**
  - **Type / role:** Cron trigger node; workflow entry point.
  - **Configuration:** `everyHour` schedule.
  - **Input/Output:** No input. Outputs to **Fetch Brand Mentions (SerpAPI)**.
  - **Failure/edge cases:** n8n instance time zone differences may shift trigger timing; missed runs possible if instance is down.

---

### 2.2 Search & Data Collection (SerpAPI)

**Overview:** Queries Google AI Mode via SerpAPI to obtain aggregated AI summaries (“text_blocks”) and individual sources (“references”) mentioning your brand.

**Nodes involved**
- **Fetch Brand Mentions (SerpAPI)**

**Node details**
- **Fetch Brand Mentions (SerpAPI)**
  - **Type / role:** `n8n-nodes-serpapi.serpApi`; external search retrieval.
  - **Configuration choices:**
    - **Operation:** `google_ai_mode`
    - **Query (`q`):** `site:reddit.com "[Your Brand Name]" reviews`
      - Intended to be customized with the actual brand name and (optionally) expanded to other sites/keywords.
  - **Credentials:** SerpAPI credential configured in n8n (API key stored in Credentials).
  - **Input/Output:** Input from **Schedule: Every Hour**. Output to **Parse Search Results**.
  - **Failure/edge cases:**
    - SerpAPI quota/rate limits (429), invalid key (401/403), intermittent network failures.
    - Google AI Mode response shape can vary; fields like `text_blocks`, `references`, or `search_metadata.google_ai_mode_url` may be missing.

---

### 2.3 Parsing & Normalization

**Overview:** Transforms the SerpAPI response into a list of standardized “insight” items for downstream AI analysis (paragraph summaries, list points, and site references).

**Nodes involved**
- **Parse Search Results**

**Node details**
- **Parse Search Results**
  - **Type / role:** Code node (JavaScript); data extraction + itemization.
  - **Configuration choices (interpreted):**
    - Reads from each incoming item:
      - `search_metadata.google_ai_mode_url` (optional)
      - `text_blocks` (optional array)
      - `references` (optional array)
    - Produces many output items, each shaped like:
      - `source` (e.g., `"SerpAPI AI Mode"` or `"Reference"`)
      - `site` (Reddit / Glassdoor / AmbitionBox / Clutch / Other / Aggregated)
      - `title`, `text`, `url`
      - `google_ai_mode_url`
      - `category` (overview, key_insight, public_discussion, employee_review, client_review)
      - `is_reddit` boolean
  - **Key logic highlights:**
    - **text_blocks**
      - Paragraph blocks become “Summary Insight” items.
      - List blocks emit one item per list entry (“Key Point”).
    - **references**
      - Classifies platform by substring match in `ref.source`:
        - reddit / glassdoor / ambitionbox / clutch → else Other
      - Assigns category based on site, flags `is_reddit`.
  - **Input/Output:** Input from **Fetch Brand Mentions (SerpAPI)**. Output to **AI Risk Analyzer** (one item per insight).
  - **Failure/edge cases:**
    - If SerpAPI returns unexpected structures, some insights may be missed but code is defensive (`|| []` / optional chaining).
    - Potentially large number of items per run → increases AI calls and cost.
    - If `ref.source` is missing, it will be treated as “Other”.

---

### 2.4 AI Risk Detection (Azure OpenAI via LangChain Agent)

**Overview:** For each insight item, the agent prompts Azure OpenAI to produce strict JSON classification: sentiment, risk detection, risk level, reason, source platform, and whether immediate action is needed.

**Nodes involved**
- **AI Risk Analyzer**
- **Azure OpenAI Model**

**Node details**
- **AI Risk Analyzer**
  - **Type / role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates LLM call(s) using the connected language model.
  - **Configuration choices:**
    - **Prompt content:** Injects the full current item as JSON via:
      - `{{ JSON.stringify($json, null, 2) }}`
    - Requires the model to respond **STRICTLY** in a specified JSON schema:
      - `sentiment` (Positive/Neutral/Negative)
      - `risk_detected` boolean
      - `risk_level` (Low/Medium/High)
      - `reason` short (1–2 lines)
      - `source_platform` classification
      - `immediate_action_required` boolean
    - **System message:** Sets role and risk definitions; instructs conservative classification; forbids hallucination; forbids extra text outside JSON.
    - **Prompt type:** “define” (uses the node’s defined prompt/system message).
  - **Connections:**
    - Main input from **Parse Search Results**
    - LLM input from **Azure OpenAI Model** via `ai_languageModel` connection
    - Main output to **Clean AI Output**
  - **Failure/edge cases:**
    - Model may still return non-JSON (extra prose, code fences, trailing commas) → downstream parsing needed (handled later).
    - Azure OpenAI rate limits/timeouts; long prompts if insights are large.
    - If SerpAPI text contains unusual characters, it can degrade model formatting.

- **Azure OpenAI Model**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`; provides chat LLM for the agent.
  - **Configuration choices:**
    - Model: `gpt-4o-mini`
  - **Credentials:** Azure OpenAI credential (endpoint, deployment/model mapping, API key).
  - **Connections:** Provides the language model to **AI Risk Analyzer**.
  - **Failure/edge cases:** Misconfigured deployment name/model mapping, Azure content filtering blocks, token limits, regional outages.

---

### 2.5 Post-processing & Filtering

**Overview:** Converts the agent’s raw string output into safe structured fields; fails closed to Low risk when parsing fails; then filters to only “High” risk items.

**Nodes involved**
- **Clean AI Output**
- **Filter: High Risk Only**

**Node details**
- **Clean AI Output**
  - **Type / role:** Code node (JavaScript); JSON parsing + schema normalization.
  - **Configuration choices (interpreted):**
    - Reads `item.json.output` (expects a JSON string from the agent).
    - `JSON.parse(raw)` then maps to normalized object:
      - Default sentiment `"Unknown"`, risk_level `"Low"`, platform `"Other"`.
      - Forces booleans with `Boolean(...)`.
    - On parse failure, outputs:
      - `risk_level: "Low"`
      - `reason: "Failed to parse AI response"`
  - **Input/Output:** Input from **AI Risk Analyzer**. Output to **Filter: High Risk Only**.
  - **Failure/edge cases:**
    - If the agent output is not stored under `output` (node behavior/version differences), parsing will yield empty results.
    - If `output` is already an object (not string), code currently skips it (`typeof raw !== "string"` → `continue`), resulting in dropped items.

- **Filter: High Risk Only**
  - **Type / role:** IF node; gatekeeper for escalation.
  - **Condition:** String equals:
    - `value1 = {{$json["risk_level"]}}`
    - `value2 = "High"`
  - **Input/Output:** Input from **Clean AI Output**.
    - True branch outputs to:
      - **Send Google Chat Alert**
      - **Create Asana Crisis Task**
  - **Failure/edge cases:**
    - Case sensitivity: only exact `"High"` matches. If model returns `"HIGH"` or `"high"`, nothing escalates.
    - If `risk_level` missing, defaults from prior step matter.

---

### 2.6 Crisis Response

**Overview:** For each High-risk item, notify a Google Chat space and create an urgent Asana task.

**Nodes involved**
- **Send Google Chat Alert**
- **Create Asana Crisis Task**

**Node details**
- **Send Google Chat Alert**
  - **Type / role:** Google Chat node; sends message into a space.
  - **Configuration choices:**
    - **Authentication:** OAuth2
    - **Space ID:** `={{ $json.space }}`
      - **Important:** The workflow does not set `$json.space` anywhere upstream. As-is, this will likely fail unless the incoming item already contains `space`.
    - **Message UI:** Empty (no explicit message text configured).
  - **Credentials:** Google Chat OAuth2 credential.
  - **Input/Output:** Input from **Filter: High Risk Only** (true path). No downstream nodes.
  - **Failure/edge cases:**
    - Missing/invalid `spaceId` → request error.
    - No message text configured → may send blank or fail depending on node requirements/version.
    - OAuth scopes/consent not sufficient to post to the space.

- **Create Asana Crisis Task**
  - **Type / role:** Asana node; creates a task for incident handling.
  - **Configuration choices:**
    - **Authentication:** OAuth2
    - **Workspace:** `1212551193156936`
    - **Task name:** `🚨 Social Crisis Detected — Immediate Action Required`
    - Other properties not set (no description, assignee, project, due date, priority fields).
  - **Credentials:** Asana OAuth2 credential.
  - **Input/Output:** Input from **Filter: High Risk Only** (true path). No downstream nodes.
  - **Failure/edge cases:**
    - Workspace ID wrong/not accessible to the OAuth user.
    - Task created without enough context unless you add description fields (risk reason, source URL, excerpt).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## 🛡️ Brand Crisis & Reputation Monitor / How it works + Setup steps (SerpAPI, Azure OpenAI, Google Chat, Asana, customize query, test before hourly schedule) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## ⏰ Automated Trigger Runs hourly to continuously monitor brand mentions across social platforms and review sites. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## 🔍 Search & Data Collection Fetches brand mentions from Reddit and review sites using Google AI Mode, then parses all insights, references, and key points into structured data. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## 🤖 AI Risk Detection Analyzes each mention for sentiment, reputation risk, and crisis indicators using Azure OpenAI. Outputs clean JSON with risk levels and action recommendations. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## 🚨 Crisis Response When high-risk issues are detected, sends immediate alerts to Google Chat and creates priority tasks in Asana for rapid team response. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## 🔐 Credentials & Security Use OAuth2 for Google Chat and Asana. Store SerpAPI and Azure OpenAI keys securely in n8n credentials. Never hardcode tokens in the workflow. |
| Schedule: Every Hour | n8n-nodes-base.cron | Hourly trigger | — | Fetch Brand Mentions (SerpAPI) | ## ⏰ Automated Trigger Runs hourly to continuously monitor brand mentions across social platforms and review sites. |
| Fetch Brand Mentions (SerpAPI) | n8n-nodes-serpapi.serpApi | Google AI Mode search retrieval | Schedule: Every Hour | Parse Search Results | ## 🔍 Search & Data Collection Fetches brand mentions from Reddit and review sites using Google AI Mode, then parses all insights, references, and key points into structured data. |
| Parse Search Results | n8n-nodes-base.code | Normalize SerpAPI response into per-insight items | Fetch Brand Mentions (SerpAPI) | AI Risk Analyzer | ## 🔍 Search & Data Collection Fetches brand mentions from Reddit and review sites using Google AI Mode, then parses all insights, references, and key points into structured data. |
| AI Risk Analyzer | @n8n/n8n-nodes-langchain.agent | LLM agent to classify risk | Parse Search Results | Clean AI Output | ## 🤖 AI Risk Detection Analyzes each mention for sentiment, reputation risk, and crisis indicators using Azure OpenAI. Outputs clean JSON with risk levels and action recommendations. |
| Azure OpenAI Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI chat model provider | — (model connection) | AI Risk Analyzer (ai_languageModel) | ## 🤖 AI Risk Detection Analyzes each mention for sentiment, reputation risk, and crisis indicators using Azure OpenAI. Outputs clean JSON with risk levels and action recommendations. |
| Clean AI Output | n8n-nodes-base.code | Parse/clean agent output into safe JSON fields | AI Risk Analyzer | Filter: High Risk Only | ## 🤖 AI Risk Detection Analyzes each mention for sentiment, reputation risk, and crisis indicators using Azure OpenAI. Outputs clean JSON with risk levels and action recommendations. |
| Filter: High Risk Only | n8n-nodes-base.if | Escalation gate (High only) | Clean AI Output | Send Google Chat Alert; Create Asana Crisis Task | ## 🚨 Crisis Response When high-risk issues are detected, sends immediate alerts to Google Chat and creates priority tasks in Asana for rapid team response. |
| Send Google Chat Alert | n8n-nodes-base.googleChat | Post crisis alert to Google Chat | Filter: High Risk Only | — | ## 🚨 Crisis Response When high-risk issues are detected, sends immediate alerts to Google Chat and creates priority tasks in Asana for rapid team response. |
| Create Asana Crisis Task | n8n-nodes-base.asana | Create incident task in Asana | Filter: High Risk Only | — | ## 🚨 Crisis Response When high-risk issues are detected, sends immediate alerts to Google Chat and creates priority tasks in Asana for rapid team response. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Brand Crisis & Reputation Monitor** (or your preferred name).
   - (Optional) Add sticky notes matching the sections: Trigger, Search, AI Risk Detection, Crisis Response, Credentials/Security.

2. **Add trigger**
   - Add node: **Cron**
   - Name: **Schedule: Every Hour**
   - Set **Mode** to **Every Hour**
   - This is the start node.

3. **Add SerpAPI search node**
   - Add node: **SerpAPI**
   - Name: **Fetch Brand Mentions (SerpAPI)**
   - Operation: **google_ai_mode**
   - Query (`q`): `site:reddit.com "[Your Brand Name]" reviews`
     - Replace `[Your Brand Name]` with your real brand string.
   - Credentials:
     - Create SerpAPI credentials in n8n and select them here.
   - Connect: **Schedule: Every Hour → Fetch Brand Mentions (SerpAPI)**

4. **Add parsing Code node**
   - Add node: **Code** (JavaScript)
   - Name: **Parse Search Results**
   - Paste the logic that:
     - Extracts `search_metadata.google_ai_mode_url`, `text_blocks`, `references`
     - Emits one n8n item per paragraph/list entry/reference with fields:
       - `source, site, title, text, url, google_ai_mode_url, category, is_reddit`
   - Connect: **Fetch Brand Mentions (SerpAPI) → Parse Search Results**

5. **Add Azure OpenAI model node**
   - Add node: **Azure OpenAI Chat Model** (LangChain)
   - Name: **Azure OpenAI Model**
   - Model: `gpt-4o-mini` (or your Azure deployment equivalent)
   - Credentials:
     - Configure Azure OpenAI credentials (resource endpoint, API key, deployment/model mapping as required by your n8n version).

6. **Add LangChain Agent node**
   - Add node: **AI Agent** (LangChain Agent)
   - Name: **AI Risk Analyzer**
   - Prompt: include the “tasks” list and force **STRICT JSON** output.
   - Include the current item in the prompt with an expression like:
     - `{{ JSON.stringify($json, null, 2) }}`
   - System message: set the crisis analyst role + risk level definitions + “JSON only”.
   - Connect the model to the agent:
     - **Azure OpenAI Model (ai_languageModel) → AI Risk Analyzer**
   - Connect data flow:
     - **Parse Search Results → AI Risk Analyzer**

7. **Add cleaning Code node**
   - Add node: **Code** (JavaScript)
   - Name: **Clean AI Output**
   - Implement:
     - Read the agent’s raw string output (commonly `item.json.output`)
     - `JSON.parse`
     - Normalize to:
       - `sentiment, risk_detected, risk_level, reason, source_platform, immediate_action_required`
     - On parse failure, set `risk_level` to **Low** and reason to parsing failure.
   - Connect: **AI Risk Analyzer → Clean AI Output**

8. **Add filter**
   - Add node: **IF**
   - Name: **Filter: High Risk Only**
   - Condition:
     - String: `{{$json.risk_level}}` equals `High`
   - Connect: **Clean AI Output → Filter: High Risk Only**

9. **Add Google Chat alert**
   - Add node: **Google Chat**
   - Name: **Send Google Chat Alert**
   - Authentication: **OAuth2**
   - Space ID:
     - Either hardcode a space ID, or set it via upstream data.
     - The provided workflow uses: `={{ $json.space }}` (ensure you actually produce `space` upstream if you keep this).
   - Message:
     - Configure message text (recommended) using fields from **Clean AI Output** (risk_level, reason, source_platform).
   - Credentials:
     - Create/select Google Chat OAuth2 credentials with posting permission to that space.
   - Connect (true branch): **Filter: High Risk Only → Send Google Chat Alert**

10. **Add Asana task creation**
    - Add node: **Asana**
    - Name: **Create Asana Crisis Task**
    - Authentication: **OAuth2**
    - Workspace: select workspace or paste ID (example in workflow: `1212551193156936`)
    - Task name: `🚨 Social Crisis Detected — Immediate Action Required`
    - (Recommended) Add description/custom fields using AI outputs (reason, source_platform, url if you keep it).
    - Credentials:
      - Create/select Asana OAuth2 credentials with access to the workspace.
    - Connect (true branch): **Filter: High Risk Only → Create Asana Crisis Task**

11. **Test and enable**
    - Run manually once to verify:
      - SerpAPI returns `text_blocks`/`references`
      - Code parsing produces multiple items
      - Agent output is parseable JSON
      - Filter catches High risk
      - Google Chat and Asana actions succeed
    - Then activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Use OAuth2 for Google Chat and Asana. Store SerpAPI and Azure OpenAI keys securely in n8n credentials. Never hardcode tokens in the workflow. | Credentials & Security sticky note |
| Customize the SerpAPI query with your brand name and test manually before enabling the hourly schedule. | Setup guidance sticky note |
| Workflow behavior: alerts/tasks only trigger when `risk_level` equals exactly `High`. | Filtering logic |

