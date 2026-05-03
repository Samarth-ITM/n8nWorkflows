Monitor brand reputation and detect crises with GPT-4, Slack and Gmail

https://n8nworkflows.xyz/workflows/monitor-brand-reputation-and-detect-crises-with-gpt-4--slack-and-gmail-12283


# Monitor brand reputation and detect crises with GPT-4, Slack and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow monitors a brand’s online reputation every 15 minutes by collecting mentions from multiple sources (news, social media, reviews, forums), normalizing the data, running a GPT-4–based analysis (sentiment, trends, crisis detection), and then triggering crisis alerts (Slack + Gmail) and logging results to Google Sheets.

**Target use cases:**
- Early detection of PR crises and negative sentiment spikes
- Continuous brand perception tracking across fragmented channels
- Automated escalation to a crisis-response team + reporting dashboard

### 1.1 Scheduling & Configuration
Runs periodically and centralizes all configurable inputs (brand name, API URLs, thresholds).

### 1.2 Data Collection (4 parallel sources)
Fetches brand mentions from four endpoints (news, social, reviews, forums).

### 1.3 Aggregation & Normalization
Merges all fetched datasets and maps varying payload shapes into a consistent schema.

### 1.4 AI Analysis (Sentiment, Trends, Crisis)
Uses an n8n LangChain Agent with an OpenAI Chat Model (GPT-4o) and a structured output parser to produce metrics + crisis flags.

### 1.5 Crisis Escalation & Reporting
If crisis detected, sends Slack + Gmail alerts. Regardless, prepares data and appends to a Google Sheets “monitoring dashboard”.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Configuration

**Overview:** Triggers the workflow on a fixed interval and defines runtime configuration values used by all downstream nodes (brand name, endpoints, and thresholds).

**Nodes involved:**
- Schedule Monitoring
- Workflow Configuration

#### Node: Schedule Monitoring
- **Type / role:** `Schedule Trigger` — workflow entrypoint; runs periodically.
- **Configuration choices:** Runs every **15 minutes**.
- **Input/Output:** No inputs (trigger). Output goes to **Workflow Configuration**.
- **Version-specific notes:** TypeVersion **1.3** (standard schedule trigger behavior).
- **Potential failures / edge cases:** Minimal; missed executions can happen if n8n instance is down or paused.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central config object for reuse.
- **Configuration choices (interpreted):**
  - `brandName`: placeholder string for your brand
  - `newsApiUrl`, `socialApiUrl`, `reviewsApiUrl`, `forumsApiUrl`: placeholder endpoint URLs
  - `crisisThreshold`: `0.7` (present but not used anywhere downstream)
  - `negativeThreshold`: `0.6` (present but not used anywhere downstream)
  - **Include Other Fields:** enabled (passes through any existing fields)
- **Key expressions/variables:** Downstream nodes reference values via:
  - `$('Workflow Configuration').first().json.brandName`
  - `$('Workflow Configuration').first().json.<apiUrl>`
- **Input/Output:** Receives trigger item; fans out to the four HTTP request nodes in parallel.
- **TypeVersion:** **3.4**
- **Potential failures / edge cases:**
  - Placeholders not replaced → HTTP nodes will fail with invalid URL / missing query.
  - Thresholds are configured but not enforced by logic (crisis relies entirely on AI output).

**Sticky note coverage (context):**
- “How It Works” + “Setup Steps” + “Prerequisites/Use Cases/Customization/Benefits” notes describe the intended architecture and setup (see tables below).

---

### Block 2 — Data Collection (Parallel HTTP Requests)

**Overview:** Pulls recent brand mentions from four external systems. Each request uses the configured API URL and passes the brand name in query parameters.

**Nodes involved:**
- Fetch News Articles
- Fetch Social Media Mentions
- Fetch Online Reviews
- Fetch Forum Discussions

#### Node: Fetch News Articles
- **Type / role:** `HTTP Request` — query news monitoring endpoint.
- **Configuration choices:**
  - **URL:** `={{ $('Workflow Configuration').first().json.newsApiUrl }}`
  - **Response format:** JSON
  - **Query params:**
    - `q = brandName`
    - `sortBy = publishedAt`
- **Input/Output:** Input from **Workflow Configuration**; output to **Combine All Sources** (input index 0).
- **TypeVersion:** **4.3**
- **Potential failures / edge cases:**
  - Auth not configured (node has no explicit auth settings) → depends on whether endpoint is public, IP-allowlisted, or expects headers.
  - API rate limits (given 15-minute schedule).
  - Response shape mismatches (e.g., results wrapped in `articles[]`) can prevent downstream normalization (since normalization expects fields directly on `$json`).

#### Node: Fetch Social Media Mentions
- **Type / role:** `HTTP Request` — query social monitoring endpoint.
- **Configuration choices:**
  - **URL:** `socialApiUrl`
  - **Response:** JSON
  - **Query params:**
    - `query = brandName`
    - `type = recent`
- **Input/Output:** From **Workflow Configuration**; to **Combine All Sources** (input index 1).
- **TypeVersion:** **4.3**
- **Potential failures / edge cases:** Same as above; social APIs often require OAuth/bearer token and enforce strict rate limits.

#### Node: Fetch Online Reviews
- **Type / role:** `HTTP Request` — query reviews aggregator endpoint.
- **Configuration choices:**
  - **URL:** `reviewsApiUrl`
  - **Query params:** `business = brandName`
  - **Response:** JSON
- **Input/Output:** From **Workflow Configuration**; to **Combine All Sources** (another merge input; in practice “combineAll” gathers all inputs).
- **TypeVersion:** **4.3**
- **Potential failures / edge cases:** Reviews data often paginated; this workflow does not handle pagination or deduplication.

#### Node: Fetch Forum Discussions
- **Type / role:** `HTTP Request` — query forums/Reddit-like endpoint.
- **Configuration choices:**
  - **URL:** `forumsApiUrl`
  - **Query params:**
    - `q = brandName`
    - `sort = new`
  - **Response:** JSON
- **Input/Output:** From **Workflow Configuration**; to **Combine All Sources**.
- **TypeVersion:** **4.3**
- **Potential failures / edge cases:** Similar; forum APIs commonly return nested arrays and require mapping.

**Important integration note (applies to all four HTTP nodes):**
- The workflow assumes that each HTTP node emits **items representing mentions**, not a single item containing an array. If each API returns `{ items: [...] }` (single item), you likely need an `Item Lists` / `Split Out` step before merging and normalization.

**Sticky note coverage (context):**
- “Data Collection” note explains multi-channel coverage rationale.

---

### Block 3 — Aggregation & Normalization

**Overview:** Combines the four streams into a single unified stream and standardizes key fields (content/title/url/date/author/source) so the AI agent can consume them consistently.

**Nodes involved:**
- Combine All Sources
- Normalize Data Structure

#### Node: Combine All Sources
- **Type / role:** `Merge` — aggregate parallel inputs.
- **Configuration choices:**
  - **Mode:** `combine`
  - **CombineBy:** `combineAll` (collect everything)
- **Input/Output:** Inputs from the 4 HTTP nodes; output to **Normalize Data Structure**.
- **TypeVersion:** **3.2**
- **Potential failures / edge cases:**
  - If upstream nodes output different item counts, “combine” modes can produce unexpected structures depending on mode; `combineAll` generally bundles streams but may yield non-ideal item grouping for later per-mention processing.
  - If any upstream node errors and workflow stops, merge may never run (unless you enable “Continue On Fail” upstream).

#### Node: Normalize Data Structure
- **Type / role:** `Set` — maps diverse payload fields into a normalized mention schema.
- **Configuration choices (field mapping):**
  - `source = $json.source || 'unknown'`
  - `content = $json.text || $json.content || $json.description || $json.body || ''`
  - `title = $json.title || $json.subject || ''`
  - `url = $json.url || $json.link || ''`
  - `publishedAt = $json.publishedAt || $json.created_at || $json.timestamp || new Date().toISOString()`
  - `author = $json.author || $json.username || 'anonymous'`
  - **Include Other Fields:** enabled
- **Input/Output:** From **Combine All Sources**; to **Sentiment & Trend Analysis Agent**.
- **TypeVersion:** **3.4**
- **Potential failures / edge cases:**
  - If upstream data is nested (e.g., `$json.articles[0].title`) none of these fallbacks will hit → content becomes empty and AI quality degrades.
  - `new Date()` in expressions uses server timezone/locale; `toLocaleDateString()` later is locale-dependent.

**Sticky note coverage (context):**
- “Analysis & Scoring” note explains merge/normalize + AI analysis rationale.

---

### Block 4 — AI Analysis (Agent + LLM + Structured Output)

**Overview:** Sends all normalized mentions to a LangChain Agent that uses GPT-4o and enforces a structured JSON output format (metrics, crisis boolean, trends, recommendations). The structured output becomes the canonical data used for alerts and logging.

**Nodes involved:**
- Sentiment & Trend Analysis Agent
- OpenAI GPT-4
- Structured Analysis Output

#### Node: OpenAI GPT-4
- **Type / role:** `LM Chat OpenAI` — provides the chat model used by the agent.
- **Configuration choices:**
  - **Model:** `gpt-4o`
  - **Temperature:** `0.3` (more deterministic, analytical outputs)
  - Built-in tools: none enabled
- **Credentials:** `openAiApi` (OpenAI account)
- **Connections:** Feeds into the Agent via `ai_languageModel`.
- **TypeVersion:** **1.3**
- **Potential failures / edge cases:**
  - Invalid/expired OpenAI key, insufficient quota, model availability, or policy blocks.
  - Large payload risk: the agent prompt includes `JSON.stringify($input.all())`; with high mention volume, you may hit token limits.

#### Node: Structured Analysis Output
- **Type / role:** `Structured Output Parser` — enforces a JSON schema for the agent’s response.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `overallSentiment` (string)
    - `sentimentScore` (-1..1 number)
    - mention counts
    - `isCrisis` (boolean)
    - `crisisLevel` (string)
    - arrays for reasons/trends/influencers/recommendations/issues
- **Connections:** Connected to the Agent as `ai_outputParser`.
- **TypeVersion:** **1.3**
- **Potential failures / edge cases:**
  - If the model returns non-conforming JSON, the parser may fail and stop execution.
  - The schema does not mark fields as “required” explicitly; depending on parser behavior/version, missing fields may still error or produce nulls.

#### Node: Sentiment & Trend Analysis Agent
- **Type / role:** `LangChain Agent` — orchestrates prompt, model, and structured parsing.
- **Configuration choices:**
  - **Prompt (`text`):**  
    `Analyze the following brand mentions... {{ JSON.stringify($input.all()) }}`
    - Uses **all incoming items** in a single call.
  - **System message:** Defines six tasks including per-mention sentiment/confidence, trend detection, crisis detection criteria, metrics, influencers, and recommendations.
  - **Prompt type:** “define”
  - **Output parser:** enabled (`hasOutputParser: true`) and linked to **Structured Analysis Output**
  - **Language model:** linked to **OpenAI GPT-4**
- **Input/Output:** Input from **Normalize Data Structure**; outputs to **Check for Crisis** and **Prepare Dashboard Data** (parallel).
- **TypeVersion:** **3.1**
- **Potential failures / edge cases:**
  - Token overflow if too many mentions are included (because `$input.all()` packs everything).
  - Hallucinated metrics: counts and “confidence scores” are requested, but schema doesn’t include per-mention objects; the agent may compress or approximate.
  - The workflow-defined thresholds (`crisisThreshold`, `negativeThreshold`) are not injected into the agent prompt; crisis flagging is subjective to the model unless you add those values into the prompt and/or add deterministic rules after parsing.

---

### Block 5 — Crisis Escalation & Reporting

**Overview:** Routes execution based on the AI’s `isCrisis` flag. If true, it sends a formatted alert to Slack and an HTML email via Gmail. Separately, it appends analysis records to Google Sheets after adding timestamps and brand info.

**Nodes involved:**
- Check for Crisis
- Send Crisis Alert to Slack
- Send Crisis Alert Email
- Prepare Dashboard Data
- Log to Monitoring Dashboard

#### Node: Check for Crisis
- **Type / role:** `IF` — conditional routing.
- **Condition:** `{{ $json.isCrisis }}` is boolean **true**.
- **Input/Output:** Input from **Sentiment & Trend Analysis Agent**; “true” output goes to Slack and Gmail nodes. (No “false” path connected.)
- **TypeVersion:** **2.3**
- **Potential failures / edge cases:**
  - If `isCrisis` is missing or string `"true"` rather than boolean, the condition may fail depending on “loose” validation (it is set to loose, which helps).
  - No else-path means non-crisis runs will not notify; that’s intended.

#### Node: Send Crisis Alert to Slack
- **Type / role:** `Slack` — posts crisis alert to a channel.
- **Configuration choices:**
  - **Authentication:** OAuth2 (Slack)
  - **Target:** channel ID/name placeholder
  - **Message:** Markdown-like alert including:
    - crisis level, overall sentiment, score, totals
    - bullet lists generated via JS expressions:
      - `crisisReasons.map(...)`
      - `topNegativeIssues.map(...)`
      - `recommendations.map(...)`
- **Credentials:** `slackOAuth2Api`
- **Input/Output:** Triggered only on crisis path from **Check for Crisis**.
- **TypeVersion:** **2.4**
- **Potential failures / edge cases:**
  - Channel ID invalid, bot not in channel, missing scopes (e.g., `chat:write`).
  - If arrays are undefined/null, `.map` will throw (expression error) and the node will fail. You may want guards like `($json.crisisReasons ?? [])`.

#### Node: Send Crisis Alert Email
- **Type / role:** `Gmail` — sends HTML email alert.
- **Configuration choices:**
  - **To:** placeholder recipient list
  - **Subject:** includes `crisisLevel`
  - **HTML body:** uses list rendering:
    - `crisisReasons.map(...)`
    - `topNegativeIssues.map(...)`
    - `recommendations.map(...)`
- **Credentials:** `gmailOAuth2`
- **Input/Output:** Triggered only on crisis path from **Check for Crisis**.
- **TypeVersion:** **2.2**
- **Potential failures / edge cases:**
  - Gmail OAuth expired / missing permissions.
  - Same `.map` null-array risk as Slack node.
  - Sending limits/quota on Gmail account.

#### Node: Prepare Dashboard Data
- **Type / role:** `Set` — adds timestamp/brand context before logging.
- **Configuration choices:**
  - `timestamp = new Date().toISOString()`
  - `date = new Date().toLocaleDateString()`
  - `time = new Date().toLocaleTimeString()`
  - `brandName = $('Workflow Configuration').first().json.brandName`
  - Include other fields (keeps the AI analysis fields)
- **Input/Output:** From **Sentiment & Trend Analysis Agent**; to **Log to Monitoring Dashboard**.
- **TypeVersion:** **3.4**
- **Potential failures / edge cases:**
  - Locale-dependent date/time formats; may complicate sheet sorting/filtering. ISO-only is usually best.

#### Node: Log to Monitoring Dashboard
- **Type / role:** `Google Sheets` — appends a row for each execution/analysis.
- **Configuration choices:**
  - **Operation:** append
  - **Document ID:** placeholder
  - **Sheet name:** placeholder
  - **Column mapping:** auto-map input data (relies on sheet headers matching JSON keys)
- **Credentials:** `googleSheetsOAuth2Api`
- **Input/Output:** Input from **Prepare Dashboard Data**; terminal node.
- **TypeVersion:** **4.7**
- **Potential failures / edge cases:**
  - Sheet headers not matching keys → columns may not populate as expected.
  - Permissions issues on the document.
  - If arrays are present (reasons/trends/etc.), Google Sheets may store them as JSON-like strings; consider joining arrays into strings before append.

**Sticky note coverage (context):**
- “Crisis Response” note explains why IF + alerts exist.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Monitoring | Schedule Trigger | Periodic workflow trigger (every 15 minutes) | — | Workflow Configuration | ## **Data Collection**  \n**What:** Schedule triggers monitoring; four HTTP nodes fetch brand mentions from news APIs  \n**Why:** Multi-channel collection ensures full coverage of reputation risks |
| Workflow Configuration | Set | Centralized configuration (brand, endpoints, thresholds) | Schedule Monitoring | Fetch News Articles; Fetch Social Media Mentions; Fetch Online Reviews; Fetch Forum Discussions | ## How It Works  \nThis workflow automates brand reputation monitoring by analyzing sentiment across news, social media, reviews, and forums using AI-powered trend detection. Designed for PR teams, brand managers, marketing directors, and crisis communication specialists requiring real-time awareness of reputation threats before they escalate.The template solves the challenge of manually tracking brand mentions across fragmented channels, news outlets, Twitter, Instagram, review sites, Reddit, industry forums, then identifying emerging crises hidden in sentiment shifts and volume spikes.Scheduled execution triggers four parallel HTTP nodes fetching data from news APIs, social media monitoring services, review aggregators, and forum discussion platforms. Merge node combines all sources, then normalization ensures consistent data structure. OpenAI GPT-4 with structured output parsing performs sophisticated sentiment analysis and trend detection  |
| Fetch News Articles | HTTP Request | Fetch news mentions for brand | Workflow Configuration | Combine All Sources | ## **Data Collection**  \n**What:** Schedule triggers monitoring; four HTTP nodes fetch brand mentions from news APIs  \n**Why:** Multi-channel collection ensures full coverage of reputation risks |
| Fetch Social Media Mentions | HTTP Request | Fetch social mentions for brand | Workflow Configuration | Combine All Sources | ## **Data Collection**  \n**What:** Schedule triggers monitoring; four HTTP nodes fetch brand mentions from news APIs  \n**Why:** Multi-channel collection ensures full coverage of reputation risks |
| Fetch Online Reviews | HTTP Request | Fetch review mentions for brand | Workflow Configuration | Combine All Sources | ## **Data Collection**  \n**What:** Schedule triggers monitoring; four HTTP nodes fetch brand mentions from news APIs  \n**Why:** Multi-channel collection ensures full coverage of reputation risks |
| Fetch Forum Discussions | HTTP Request | Fetch forum mentions for brand | Workflow Configuration | Combine All Sources | ## **Data Collection**  \n**What:** Schedule triggers monitoring; four HTTP nodes fetch brand mentions from news APIs  \n**Why:** Multi-channel collection ensures full coverage of reputation risks |
| Combine All Sources | Merge | Combine all source streams into one | Fetch News Articles; Fetch Social Media Mentions; Fetch Online Reviews; Fetch Forum Discussions | Normalize Data Structure | ## **Analysis & Scoring**  \n**What:** Merge combines streams, normalization standardizes data, then GPT-4  analyzes sentiment trends  \n**Why:** AI uncovers subtle crisis indicators missed by rule-based systems |
| Normalize Data Structure | Set | Standardize mention fields (content/title/url/date/author/source) | Combine All Sources | Sentiment & Trend Analysis Agent | ## **Analysis & Scoring**  \n**What:** Merge combines streams, normalization standardizes data, then GPT-4  analyzes sentiment trends  \n**Why:** AI uncovers subtle crisis indicators missed by rule-based systems |
| Sentiment & Trend Analysis Agent | LangChain Agent | Analyze all mentions; output structured metrics + crisis flag | Normalize Data Structure; (AI) OpenAI GPT-4; (AI) Structured Analysis Output | Check for Crisis; Prepare Dashboard Data | ## **Analysis & Scoring**  \n**What:** Merge combines streams, normalization standardizes data, then GPT-4  analyzes sentiment trends  \n**Why:** AI uncovers subtle crisis indicators missed by rule-based systems |
| OpenAI GPT-4 | OpenAI Chat Model (LangChain) | LLM used by the agent (gpt-4o) | — (AI connection) | Sentiment & Trend Analysis Agent (AI) | ## Prerequisites  \nOpenAI API key, news monitoring API access  \n## Use Cases  \nConsumer brands monitoring product launch reception and identifying quality issues early  \n## Customization  \nModify AI prompts for industry-specific crisis indicators  \n## Benefits  \nReduces crisis detection time from hours to minutes enabling damage control before viral spread |
| Structured Analysis Output | Structured Output Parser (LangChain) | Enforce JSON schema for analysis result | — (AI connection) | Sentiment & Trend Analysis Agent (AI) | ## **Analysis & Scoring**  \n**What:** Merge combines streams, normalization standardizes data, then GPT-4  analyzes sentiment trends  \n**Why:** AI uncovers subtle crisis indicators missed by rule-based systems |
| Check for Crisis | IF | Route only if `isCrisis === true` | Sentiment & Trend Analysis Agent | Send Crisis Alert to Slack; Send Crisis Alert Email | ## **Crisis Response**  \n**What:** IF node flags high-severity crises, triggering Slack and Gmail alerts  \n**Why:** Immediate alerts enable rapid response to prevent reputation damage |
| Send Crisis Alert to Slack | Slack | Post formatted crisis summary to Slack channel | Check for Crisis (true path) | — | ## **Crisis Response**  \n**What:** IF node flags high-severity crises, triggering Slack and Gmail alerts  \n**Why:** Immediate alerts enable rapid response to prevent reputation damage |
| Send Crisis Alert Email | Gmail | Email formatted crisis summary to recipients | Check for Crisis (true path) | — | ## **Crisis Response**  \n**What:** IF node flags high-severity crises, triggering Slack and Gmail alerts  \n**Why:** Immediate alerts enable rapid response to prevent reputation damage |
| Prepare Dashboard Data | Set | Add timestamps and brand name to analysis result | Sentiment & Trend Analysis Agent | Log to Monitoring Dashboard | ## Setup Steps  \n1. Configure HTTP nodes with API credentials for news monitoring service   \n2. Add OpenAI API key to Chat Model node for sentiment and trend analysis  \n3. Connect Slack workspace and specify crisis response team channel   \n4. Integrate Gmail account with PR leadership distribution list   \n5. Set up Google Sheets connection and create monitoring dashboard  |
| Log to Monitoring Dashboard | Google Sheets | Append analysis record to monitoring sheet | Prepare Dashboard Data | — | ## Setup Steps  \n1. Configure HTTP nodes with API credentials for news monitoring service   \n2. Add OpenAI API key to Chat Model node for sentiment and trend analysis  \n3. Connect Slack workspace and specify crisis response team channel   \n4. Integrate Gmail account with PR leadership distribution list   \n5. Set up Google Sheets connection and create monitoring dashboard  |
| Sticky Note | Sticky Note | Documentation/commentary | — | — | ## How It Works  \nThis workflow automates brand reputation monitoring by analyzing sentiment across news, social media, reviews, and forums using AI-powered trend detection. Designed for PR teams, brand managers, marketing directors, and crisis communication specialists requiring real-time awareness of reputation threats before they escalate.The template solves the challenge of manually tracking brand mentions across fragmented channels, news outlets, Twitter, Instagram, review sites, Reddit, industry forums, then identifying emerging crises hidden in sentiment shifts and volume spikes.Scheduled execution triggers four parallel HTTP nodes fetching data from news APIs, social media monitoring services, review aggregators, and forum discussion platforms. Merge node combines all sources, then normalization ensures consistent data structure. OpenAI GPT-4 with structured output parsing performs sophisticated sentiment analysis and trend detection  |
| Sticky Note1 | Sticky Note | Documentation/commentary | — | — | ## Prerequisites  \nOpenAI API key, news monitoring API access  \n## Use Cases  \nConsumer brands monitoring product launch reception and identifying quality issues early  \n## Customization  \nModify AI prompts for industry-specific crisis indicators  \n## Benefits  \nReduces crisis detection time from hours to minutes enabling damage control before viral spread |
| Sticky Note2 | Sticky Note | Documentation/commentary | — | — | ## Setup Steps  \n1. Configure HTTP nodes with API credentials for news monitoring service   \n2. Add OpenAI API key to Chat Model node for sentiment and trend analysis  \n3. Connect Slack workspace and specify crisis response team channel   \n4. Integrate Gmail account with PR leadership distribution list   \n5. Set up Google Sheets connection and create monitoring dashboard  |
| Sticky Note3 | Sticky Note | Documentation/commentary | — | — | ## **Crisis Response**  \n**What:** IF node flags high-severity crises, triggering Slack and Gmail alerts  \n**Why:** Immediate alerts enable rapid response to prevent reputation damage |
| Sticky Note4 | Sticky Note | Documentation/commentary | — | — | ## **Analysis & Scoring**  \n**What:** Merge combines streams, normalization standardizes data, then GPT-4  analyzes sentiment trends  \n**Why:** AI uncovers subtle crisis indicators missed by rule-based systems |
| Sticky Note5 | Sticky Note | Documentation/commentary | — | — | ## **Data Collection**  \n**What:** Schedule triggers monitoring; four HTTP nodes fetch brand mentions from news APIs  \n**Why:** Multi-channel collection ensures full coverage of reputation risks |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: *AI-Driven Brand Reputation Monitoring and Crisis Detection* (or your preferred name)
- Keep workflow **inactive** until credentials and endpoints are set.

2) **Add trigger**
- Add node: **Schedule Trigger**
- Configure: Every **15 minutes**
- Node name: **Schedule Monitoring**

3) **Add configuration node**
- Add node: **Set**
- Node name: **Workflow Configuration**
- Add fields:
  - `brandName` (string)
  - `newsApiUrl` (string)
  - `socialApiUrl` (string)
  - `reviewsApiUrl` (string)
  - `forumsApiUrl` (string)
  - `crisisThreshold` (number) = 0.7
  - `negativeThreshold` (number) = 0.6
- Enable **Include Other Fields**
- Connect: **Schedule Monitoring → Workflow Configuration**

4) **Add 4 HTTP Request nodes (parallel)**
- Add **HTTP Request** node #1 named **Fetch News Articles**
  - URL: `{{$('Workflow Configuration').first().json.newsApiUrl}}`
  - Response: JSON
  - Send Query Params: true
  - Params: `q={{brandName}}`, `sortBy=publishedAt`
- Add **HTTP Request** node #2 named **Fetch Social Media Mentions**
  - URL: `{{$('Workflow Configuration').first().json.socialApiUrl}}`
  - Params: `query={{brandName}}`, `type=recent`
- Add **HTTP Request** node #3 named **Fetch Online Reviews**
  - URL: `{{$('Workflow Configuration').first().json.reviewsApiUrl}}`
  - Params: `business={{brandName}}`
- Add **HTTP Request** node #4 named **Fetch Forum Discussions**
  - URL: `{{$('Workflow Configuration').first().json.forumsApiUrl}}`
  - Params: `q={{brandName}}`, `sort=new`
- Connect: **Workflow Configuration → each HTTP node** (4 outgoing connections)

5) **Merge all sources**
- Add node: **Merge**
- Name: **Combine All Sources**
- Mode: **Combine**
- Combine By: **Combine All**
- Connect each HTTP node output into the Merge node (4 inputs).

6) **Normalize output fields**
- Add node: **Set**
- Name: **Normalize Data Structure**
- Enable **Include Other Fields**
- Add fields with expressions:
  - `source = {{$json.source || 'unknown'}}`
  - `content = {{$json.text || $json.content || $json.description || $json.body || ''}}`
  - `title = {{$json.title || $json.subject || ''}}`
  - `url = {{$json.url || $json.link || ''}}`
  - `publishedAt = {{$json.publishedAt || $json.created_at || $json.timestamp || new Date().toISOString()}}`
  - `author = {{$json.author || $json.username || 'anonymous'}}`
- Connect: **Combine All Sources → Normalize Data Structure**

7) **Add OpenAI chat model**
- Add node: **OpenAI Chat Model** (LangChain) / “LM Chat OpenAI”
- Name: **OpenAI GPT-4**
- Model: **gpt-4o**
- Temperature: **0.3**
- Credentials: configure **OpenAI API** credential in n8n and select it here.

8) **Add structured output parser**
- Add node: **Structured Output Parser**
- Name: **Structured Analysis Output**
- Schema: paste/create a manual schema containing fields:
  - overallSentiment (string), sentimentScore (number), totalMentions/positiveMentions/negativeMentions/neutralMentions (numbers),
  - isCrisis (boolean), crisisLevel (string), crisisReasons (string[]),
  - emergingTrends (string[]), keyInfluencers (string[]), recommendations (string[]), topNegativeIssues (string[]).

9) **Add AI agent**
- Add node: **AI Agent** (LangChain Agent)
- Name: **Sentiment & Trend Analysis Agent**
- Prompt (Text): include all items, e.g.  
  `Analyze the following brand mentions... {{ JSON.stringify($input.all()) }}`
- System message: set to your analyst instructions (sentiment + trends + crisis logic).
- Connect AI components:
  - Connect **OpenAI GPT-4** to the Agent via the **AI Language Model** connection.
  - Connect **Structured Analysis Output** to the Agent via the **AI Output Parser** connection.
- Connect main flow: **Normalize Data Structure → Sentiment & Trend Analysis Agent**

10) **Add crisis decision**
- Add node: **IF**
- Name: **Check for Crisis**
- Condition: Boolean **true** on `{{$json.isCrisis}}`
- Connect: **Sentiment & Trend Analysis Agent → Check for Crisis**

11) **Add Slack alert**
- Add node: **Slack**
- Name: **Send Crisis Alert to Slack**
- Authentication: **OAuth2**
- Credentials: connect Slack OAuth2 credential (must have permission to post)
- Channel: choose channel ID/name (your crisis team channel)
- Message: build using fields from the structured output (crisisLevel, sentimentScore, arrays, etc.)
- Connect from **Check for Crisis (true)** to this Slack node.

12) **Add Gmail alert**
- Add node: **Gmail**
- Name: **Send Crisis Alert Email**
- Credentials: Gmail OAuth2 credential
- To: crisis distribution list
- Subject: include `{{$json.crisisLevel}}`
- Message: HTML body rendering reasons/issues/recommendations
- Connect from **Check for Crisis (true)** to this Gmail node.

13) **Prepare dashboard logging**
- Add node: **Set**
- Name: **Prepare Dashboard Data**
- Enable **Include Other Fields**
- Add fields:
  - `timestamp = {{new Date().toISOString()}}`
  - `date = {{new Date().toLocaleDateString()}}`
  - `time = {{new Date().toLocaleTimeString()}}`
  - `brandName = {{$('Workflow Configuration').first().json.brandName}}`
- Connect: **Sentiment & Trend Analysis Agent → Prepare Dashboard Data**

14) **Append to Google Sheets**
- Add node: **Google Sheets**
- Name: **Log to Monitoring Dashboard**
- Operation: **Append**
- Credentials: Google Sheets OAuth2
- Document ID: select your spreadsheet
- Sheet name: select/create your monitoring sheet
- Column mapping: **Auto-map**
- Connect: **Prepare Dashboard Data → Log to Monitoring Dashboard**

15) **Final checks**
- Replace all placeholder values (URLs, recipients, Slack channel, Sheet ID/name).
- Run once manually to validate:
  - HTTP outputs are in “items” form (not nested arrays)
  - Agent output conforms to schema
  - Slack/Gmail `.map()` expressions don’t break on missing arrays
- Activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How It Works” note describing the end-to-end architecture: scheduled execution → 4 parallel fetches → merge → normalization → GPT-4 analysis with structured parsing | In-workflow sticky note (How It Works) |
| Prerequisites: OpenAI API key + news monitoring API access; customization: modify prompts for industry-specific crisis indicators; benefit: reduces detection time | In-workflow sticky note (Prerequisites/Use Cases/Customization/Benefits) |
| Setup steps: configure HTTP nodes + OpenAI + Slack + Gmail + Google Sheets | In-workflow sticky note (Setup Steps) |
| Crisis Response rationale: IF node flags crises, triggers Slack/Gmail for rapid response | In-workflow sticky note (Crisis Response) |
| Analysis & Scoring rationale: merge+normalize+AI to detect subtle indicators | In-workflow sticky note (Analysis & Scoring) |
| Data Collection rationale: multi-channel ingestion for full coverage | In-workflow sticky note (Data Collection) |