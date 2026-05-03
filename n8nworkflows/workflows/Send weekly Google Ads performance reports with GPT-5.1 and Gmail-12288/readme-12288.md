Send weekly Google Ads performance reports with GPT-5.1 and Gmail

https://n8nworkflows.xyz/workflows/send-weekly-google-ads-performance-reports-with-gpt-5-1-and-gmail-12288


# Send weekly Google Ads performance reports with GPT-5.1 and Gmail

## 1. Workflow Overview

**Workflow name:** AI-Powered Google Ads Weekly Analyst  
**Stated title:** Send weekly Google Ads performance reports with GPT-5.1 and Gmail  
**Purpose:** Every Monday, the workflow pulls Google Ads campaign performance for two consecutive 7‑day windows (last 7 days vs. prior 7 days), computes rollups (spend/clicks/impressions/conversions, plus derived CTR/CPA), asks a GPT-5.1-powered “AI Analyst” to generate a **complete HTML report**, then emails it via Gmail.

### 1.1 Scheduling / Entry
A schedule trigger starts the workflow weekly.

### 1.2 Data Collection (Google Ads API)
Two HTTP requests query Google Ads (GAQL) for:
- last 7 days (most recent week)
- last 14 days up to today (intended “previous week”, but see notes: query window mismatch)

### 1.3 Data Shaping & Summaries
Each query’s `results` array is split into items, then summarized in Code nodes into a compact JSON structure containing totals and campaign lists (top spenders, low CTR, etc.).

### 1.4 Week Comparison Packaging
The two summary objects are merged and aggregated into a single payload for the AI Agent.

### 1.5 AI Analysis (GPT-5.1 via LangChain nodes)
A LangChain Agent uses:
- **OpenAI Chat Model (gpt-5.1)**
- **Calculator tool** (available to the agent)
to generate a strictly formatted HTML report.

### 1.6 Delivery (Gmail)
The final HTML is emailed to a recipient.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling / Entry

**Overview:** Triggers the workflow every week to start data pulls.  
**Nodes involved:** Weekly Trigger, Sticky Note1, Sticky Note5 (global note)

#### Node: Weekly Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entrypoint on a schedule.
- **Configuration (interpreted):** Runs on a weekly interval, set to trigger at **day 1 (Monday)** at the default schedule time (commonly midnight in instance timezone).
- **Inputs/Outputs:** No input. Outputs to **Get Last Week Data** and **Get Previous Week Data** in parallel.
- **Version:** 1.2
- **Edge cases / failures:**
  - Timezone: “Monday midnight” depends on n8n instance timezone settings.
  - Missed executions if instance is down; behavior depends on n8n scheduling mode.

#### Sticky Note1 (applies to scheduling area)
Content:
- “## Scheduling
  - Currently triggers every monday midnight
  - Can be modified”

#### Sticky Note5 (covers the whole workflow area; content duplicated per affected nodes in the summary table)
Contains setup instructions and overall description (see Section 5).

---

### Block 2 — Data Collection (Google Ads API)

**Overview:** Pulls campaign metrics from Google Ads using GAQL via HTTP Request nodes authenticated with Google Ads OAuth2.  
**Nodes involved:** Get Last Week Data, Get Previous Week Data, Sticky Note2, Sticky Note5

#### Node: Get Last Week Data
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Google Ads API `googleAds:search`.
- **Configuration (interpreted):**
  - Method: **POST**
  - URL: `https://googleads.googleapis.com/v20/customers/[Customer ID]/googleAds:search`
  - Body: JSON with GAQL query selecting campaign fields + metrics, filtered:
    - `segments.date BETWEEN '{{ $now.minus(7, 'days').format('yyyy-MM-dd') }}' AND '{{ $now.format('yyyy-MM-dd') }}'`
    - `campaign.status = 'ENABLED'`
    - ordered by `metrics.cost_micros DESC`
  - Headers:
    - `Content-Type: application/json`
    - `developer-token: <missing value in template>` (must be provided)
  - Auth: **predefinedCredentialType** = `googleAdsOAuth2Api`
- **Key expressions/variables:**
  - `$now.minus(7,'days').format('yyyy-MM-dd')`
  - `$now.format('yyyy-MM-dd')`
- **Inputs/Outputs:** Input from Weekly Trigger. Output JSON is expected to include a `results` array (Google Ads Search response).
- **Version:** 4.2
- **Edge cases / failures:**
  - Missing/invalid **Developer Token** ⇒ 401/403.
  - OAuth scope/consent issues ⇒ 401.
  - Invalid Customer ID formatting or no access ⇒ 403.
  - GAQL field name mismatches across API versions (v20) can break queries.
  - Date window includes “today”; if running at midnight, “today” may be partial day (depending on reporting freshness).

#### Node: Get Previous Week Data
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Google Ads API `googleAds:search`.
- **Configuration (interpreted):**
  - Method: **POST**
  - URL: `https://googleads.googleapis.com/v20/customers/[Customer ID]/googleAds:search ` (note: trailing space in the JSON)
  - Date filter:
    - `segments.date BETWEEN '{{ $now.minus(14, 'days').format('yyyy-MM-dd') }}' AND '{{ $now.format('yyyy-MM-dd') }}'`
- **Important logic note:** Despite the node name “Previous Week Data”, the query window is **last 14 days up to today**, not strictly “previous 7 days”. The Code node later labels this as `last_2_weeks_data`, and the AI prompt treats it as “PREVIOUS WEEK DATA (2 Weeks Ago)”. This mismatch can distort week-over-week comparisons unless corrected.
- **Headers/Auth:** same as above.
- **Inputs/Outputs:** Input from Weekly Trigger; output expected with `results` array.
- **Version:** 4.2
- **Edge cases / failures:**
  - Same as “Get Last Week Data”.
  - Trailing space in URL may or may not be tolerated; safest is to remove it.

#### Sticky Note2 (applies to data collection area)
Content: “## Data Collection”

---

### Block 3 — Split & Summarize (Per window)

**Overview:** Converts Google Ads `results` array into individual items, then computes totals and categorized campaign lists, returning one summary object per time window.  
**Nodes involved:** Extract Last Week Results, Calculate & Summarize Last Week, Extract Previous Week Results, Calculate & Summarize Last 2 Weeks, Sticky Note2, Sticky Note5

#### Node: Extract Last Week Results
- **Type / role:** `n8n-nodes-base.splitOut` — splits an array field into multiple items.
- **Configuration:** `fieldToSplitOut: "results"`
- **Inputs/Outputs:** Input from **Get Last Week Data**; outputs one item per element in `results`.
- **Version:** 1
- **Edge cases / failures:**
  - If `results` is missing/empty, downstream Code node may receive zero items (handled, but produces zeros/lists empty).
  - If Google Ads returns errors instead of results, this node fails.

#### Node: Calculate & Summarize Last Week
- **Type / role:** `n8n-nodes-base.code` — aggregates campaign items and enriches with CTR/CPA.
- **Configuration choices (interpreted):**
  - Reads all incoming items via `$input.all()`.
  - Maps each item into a simplified campaign object:
    - `impressions`, `clicks` parsed as integers
    - `cost` converted from micros to currency units
    - `conversions` parsed as float
    - `avg_cpc` micros to currency
  - Filters out “zero activity” campaigns (keeps if `cost > 0` OR `impressions > 0`).
  - Adds:
    - `ctr` = clicks/impressions*100 (string fixed to 2 decimals; `'0'` if no impressions)
    - `cpa` = cost/conversions (string fixed to 2 decimals; `'N/A'` if no conversions)
  - Sorts by spend descending.
  - Produces a `summary` object with:
    - totals and campaign lists:
      - `top_campaigns` top 10 by spend
      - `zero_conversions_high_spend`: conversions == 0 and cost > 50 (top 5)
      - `low_ctr_campaigns`: ctr < 1 and impressions > 100 (top 5)
      - `high_conversion_campaigns`: top 5 by conversions
  - Returns **one** item with `json.last_week_data = summary`.
- **Key variables:**
  - Uses `item.json.metrics.costMicros` and `item.json.metrics.averageCpc` (camelCase).
- **Important integration detail:** Google Ads API typically returns metric fields using the GAQL-selected names; in many clients, `cost_micros` is surfaced as `costMicros` in JSON, but this can vary. If the response contains `cost_micros` instead of `costMicros`, `cost` will become `0`, breaking spend/CPA logic.
- **Inputs/Outputs:** Input from SplitOut; output to **Merge Week Comparison Data** (input 0).
- **Version:** 2
- **Edge cases / failures:**
  - Field name mismatch (`costMicros` vs `cost_micros`) ⇒ totals become 0.
  - Large accounts: many campaigns ⇒ many items; Code node processing could be slow but typically fine.

#### Node: Extract Previous Week Results
- **Type / role:** `n8n-nodes-base.splitOut`
- **Configuration:** splits `results`.
- **Inputs/Outputs:** Input from **Get Previous Week Data**; outputs items to **Calculate & Summarize Last 2 Weeks**.
- **Version:** 1
- **Edge cases:** same as “Extract Last Week Results”.

#### Node: Calculate & Summarize Last 2 Weeks
- **Type / role:** `n8n-nodes-base.code`
- **Configuration:** Same logic as “Calculate & Summarize Last Week”, but outputs `json.last_2_weeks_data = summary`.
- **Inputs/Outputs:** Output goes to **Merge Week Comparison Data** (input 1).
- **Version:** 2
- **Edge cases:** same as above, plus the upstream date-window mismatch (14-day window) affects interpretation.

---

### Block 4 — Merge & Aggregate for AI Input

**Overview:** Combines the two summaries into a single payload structure expected by the AI Analyst prompt.  
**Nodes involved:** Merge Week Comparison Data, Aggregate, Sticky Note3, Sticky Note5

#### Node: Merge Week Comparison Data
- **Type / role:** `n8n-nodes-base.merge` — merges two incoming items/streams.
- **Configuration:** Not explicitly set in parameters (defaults apply). With two inputs connected, n8n typically merges by index when in “Combine”/default merge behavior.
- **Inputs/Outputs:**
  - Input 0: from **Calculate & Summarize Last Week**
  - Input 1: from **Calculate & Summarize Last 2 Weeks**
  - Output: to **Aggregate**
- **Version:** 3.2
- **Edge cases / failures:**
  - If one branch fails or returns no items, merge may output empty or incomplete data depending on mode.
  - If merge mode is not “Merge By Index”/compatible default, it can produce unexpected structure (verify node UI).

#### Node: Aggregate
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates fields across items.
- **Configuration (interpreted):**
  - Aggregates fields:
    - `last_week_data`
    - `last_2_weeks_data`
  - This typically produces arrays of the aggregated fields (hence the AI prompt using `[0]` indexing).
- **Inputs/Outputs:** Input from Merge; output to **AI Analyst**.
- **Version:** 1
- **Edge cases / failures:**
  - If the merge output already contains both fields in a single item, aggregation may wrap them into arrays unnecessarily; the prompt expects arrays, so this is consistent, but can be confusing.
  - If one field is missing, `$json.last_week_data[0]` expressions in the AI prompt can fail.

#### Sticky Note3 (applies to AI analysis area, including aggregation into AI)
Content: “## AI Analysis”

---

### Block 5 — AI Generation (LangChain Agent + Model + Tool)

**Overview:** Uses a LangChain agent with GPT-5.1 to transform the summarized metrics into a decisive, executive-ready HTML performance review.  
**Nodes involved:** OpenAI Chat Model, Calculator, AI Analyst, Sticky Note3, Sticky Note5

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat LLM backend to the agent.
- **Configuration (interpreted):**
  - Model: `gpt-5.1`
  - Options: default (no temperature/top_p shown)
- **Credentials:** OpenAI API credential (template shows “Dummy”).
- **Inputs/Outputs:** Connected to **AI Analyst** via `ai_languageModel`.
- **Version:** 1.2
- **Edge cases / failures:**
  - Invalid API key / quota exceeded.
  - Model name availability depends on your OpenAI account and n8n/OpenAI node version.

#### Node: Calculator
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCalculator` — optional arithmetic tool for the agent.
- **Configuration:** default.
- **Inputs/Outputs:** Connected to **AI Analyst** via `ai_tool`.
- **Version:** 1
- **Edge cases:** Usually none; tool may be unused if agent doesn’t call it.

#### Node: AI Analyst
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + tools + model to produce final HTML.
- **Configuration (interpreted):**
  - Prompt is fully defined in the node (“define” mode).
  - Uses many n8n expressions to inject aggregated data:
    - `{{ $json.last_week_data[0].total_spend }}`, etc.
    - `{{ JSON.stringify($json.last_week_data[0].top_campaigns, null, 2) }}` and similar for lists
  - Output requirement: **ONLY HTML**, must be a complete HTML document; includes required sections and table styling rules.
- **Inputs/Outputs:** Input from **Aggregate**; output to **Email Report to User**.
- **Version:** 2.1
- **Edge cases / failures:**
  - If `$json.last_week_data[0]` is undefined (aggregation/merge issues), expressions render empty or throw.
  - HTML may exceed email client limits if campaign lists are large; prompt tries to cap list sizes in Code nodes (top 10, top 5).
  - Prompt contains a contradictory instruction: “Keep all existing styling rules exactly as before.” There is no referenced “before” style source besides the included rules; model may hallucinate additional “existing” rules unless your prompt historically included them.

---

### Block 6 — Email Delivery

**Overview:** Emails the AI-generated HTML report via Gmail.  
**Nodes involved:** Email Report to User, Sticky Note4, Sticky Note5

#### Node: Email Report to User
- **Type / role:** `n8n-nodes-base.gmail` — sends an email.
- **Configuration (interpreted):**
  - To: `user@example.com` (placeholder)
  - Subject: `Weekly Report`
  - Message body: `={{ $json.output }}`
    - Assumes the AI Analyst node outputs a field named `output` containing the HTML.
- **Credentials:** Gmail OAuth2 (template shows “Dummy”).
- **Inputs/Outputs:** Input from **AI Analyst**; terminal node (no outputs).
- **Version:** 2.1
- **Edge cases / failures:**
  - If AI Analyst output field is not exactly `$json.output` (some agent configurations use different field names), email body will be blank.
  - Gmail OAuth token expired/revoked.
  - Gmail sending limits / “from” restrictions on some Google Workspace accounts.
  - HTML rendering differences across email clients.

#### Sticky Note4 (applies to delivery area)
Content: “## Delivery”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | Schedule Trigger | Weekly entrypoint (Monday) | — | Get Last Week Data; Get Previous Week Data | ## Scheduling<br>- Currently triggers every monday midnight<br>- Can be modified<br><br># 📊 AI-Powered Google Ads Weekly Analyst<br>Automatically analyzes your Google Ads performance every Monday and sends a comprehensive report to your inbox.<br><br>## What it does<br>- Pulls campaign data from the last 2 weeks via Google Ads API<br>- Compares week-over-week performance metrics<br>- Identifies top performers and problem campaigns<br>- Calculates key metrics: CTR, CPA, spend efficiency, conversion trends<br>- Generates AI-powered insights and prioritized recommendations using GPT-5.1<br>- Emails a professional HTML report with executive summary and action items<br><br>## Setup Required<br>1. **Google Ads Customer ID**: Replace `[Customer ID]` in both HTTP Request node URLs<br>- Format: XXX-XXX-XXXX (found in your Google Ads dashboard top-right)<br>2. **Developer Token**: Add your Google Ads Developer Token in the header parameters of both HTTP Request nodes<br>3. **Google Ads OAuth2**: Connect your Google Ads credentials<br>4. **Gmail OAuth2**: Connect your Gmail credentials for email delivery<br>5. **Email Recipient**: Update the recipient email in the "Email Report to User" node<br>6. **(Optional)** Adjust the schedule trigger timing<br><br>## Configuration<br>- **Runs:** Every Monday at midnight (configurable in Weekly Trigger node)<br>- **Analysis Period:** Last 7 days vs. previous 7 days<br>- **Report Format:** HTML email with tables, metrics, and color-coded insights |
| Get Last Week Data | HTTP Request | Query Google Ads last 7 days | Weekly Trigger | Extract Last Week Results | ## Data Collection<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Extract Last Week Results | Split Out | Split Google Ads `results` array into items | Get Last Week Data | Calculate & Summarize Last Week | ## Data Collection<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Calculate & Summarize Last Week | Code | Compute totals + derived CTR/CPA + campaign lists (last week) | Extract Last Week Results | Merge Week Comparison Data | ## Data Collection<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Get Previous Week Data | HTTP Request | Query Google Ads (currently last 14 days) | Weekly Trigger | Extract Previous Week Results | ## Data Collection<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Extract Previous Week Results | Split Out | Split Google Ads `results` array into items | Get Previous Week Data | Calculate & Summarize Last 2 Weeks | ## Data Collection<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Calculate & Summarize Last 2 Weeks | Code | Compute totals + derived CTR/CPA + campaign lists (labeled last_2_weeks_data) | Extract Previous Week Results | Merge Week Comparison Data | ## Data Collection<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Merge Week Comparison Data | Merge | Combine last_week_data + last_2_weeks_data into one stream | Calculate & Summarize Last Week; Calculate & Summarize Last 2 Weeks | Aggregate | ## AI Analysis<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Aggregate | Aggregate | Wrap/aggregate summary fields for agent prompt indexing | Merge Week Comparison Data | AI Analyst | ## AI Analysis<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| OpenAI Chat Model | LangChain OpenAI Chat Model | LLM backend (gpt-5.1) | — | AI Analyst (ai_languageModel) | ## AI Analysis<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Calculator | LangChain Calculator Tool | Tool for arithmetic if agent chooses | — | AI Analyst (ai_tool) | ## AI Analysis<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| AI Analyst | LangChain Agent | Generates complete HTML weekly performance review | Aggregate | Email Report to User | ## AI Analysis<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Email Report to User | Gmail | Send HTML report email | AI Analyst | — | ## Delivery<br><br># 📊 AI-Powered Google Ads Weekly Analyst (see setup/config note above) |
| Sticky Note1 | Sticky Note | Comment: scheduling | — | — | ## Scheduling<br>- Currently triggers every monday midnight<br>- Can be modified |
| Sticky Note2 | Sticky Note | Comment: data collection section | — | — | ## Data Collection |
| Sticky Note3 | Sticky Note | Comment: AI analysis section | — | — | ## AI Analysis |
| Sticky Note4 | Sticky Note | Comment: delivery section | — | — | ## Delivery |
| Sticky Note5 | Sticky Note | Global setup/description | — | — | # 📊 AI-Powered Google Ads Weekly Analyst (full note content as shown in table above) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: **AI-Powered Google Ads Weekly Analyst** (or your preferred name).
- Ensure workflow timezone is set appropriately in n8n settings.

2) **Add Trigger**
- Add node: **Schedule Trigger**
- Configure: Weekly → **Every 1 week** → **Monday** (and time = 00:00 if you want “midnight”).
- This will be the entry node.

3) **Add Google Ads HTTP request for “Last Week”**
- Add node: **HTTP Request**
- Method: **POST**
- URL: `https://googleads.googleapis.com/v20/customers/<CUSTOMER_ID>/googleAds:search`
- Authentication: **Predefined Credential Type** → **Google Ads OAuth2 API**
  - Create/attach credentials with access to the customer account.
- Headers:
  - `Content-Type: application/json`
  - `developer-token: <YOUR_DEVELOPER_TOKEN>`
- Body content type: **JSON**
- Body (GAQL query) uses expressions:
  - Date range: from `{{$now.minus(7,'days').format('yyyy-MM-dd')}}` to `{{$now.format('yyyy-MM-dd')}}`
  - Query selects:
    - campaign id/name/status
    - metrics: impressions, clicks, cost_micros, average_cpc, conversions, conversions_value, all_conversions, search impression share, search budget lost impression share
  - Filter: `campaign.status = 'ENABLED'`
  - Order: spend desc by `metrics.cost_micros`

4) **Add Google Ads HTTP request for “Previous Week”**
- Add another **HTTP Request** node with the same auth/headers.
- Recommended (to match the intended “previous 7 days” comparison): set date range to:
  - `BETWEEN {{$now.minus(14,'days').format('yyyy-MM-dd')}} AND {{$now.minus(7,'days').format('yyyy-MM-dd')}}`
- If you keep the template behavior, it will instead query the last 14 days up to today (not a true previous week).

5) **Split results arrays**
- Add node: **Split Out** (for last week)
  - Field to split out: `results`
  - Connect: **Last Week HTTP** → **Split Out**
- Add node: **Split Out** (for previous week)
  - Field: `results`
  - Connect: **Previous Week HTTP** → **Split Out**

6) **Summarize each period with Code nodes**
- Add node: **Code** named “Calculate & Summarize Last Week”
  - Input: the “last week” Split Out
  - Implement logic:
    - Map campaigns into simplified objects (id, name, impressions, clicks, costMicros→cost, conversions, averageCpc→avg_cpc)
    - Filter zero activity
    - Compute CTR (%) and CPA
    - Build `summary` totals and lists (top 10 by spend, zero conversions/high spend, low CTR, high conversions)
    - Return **one** item: `{ last_week_data: summary }`
- Add node: **Code** named “Calculate & Summarize Previous Week”
  - Same code, but return `{ last_2_weeks_data: summary }` (or rename to `previous_week_data` and update the AI prompt accordingly).

7) **Merge the two summaries**
- Add node: **Merge**
- Connect:
  - “Calculate & Summarize Last Week” → Merge (Input 1)
  - “Calculate & Summarize …” for previous → Merge (Input 2)
- Ensure merge mode results in a single combined item containing both fields (verify in node UI).

8) **Aggregate fields for the agent prompt**
- Add node: **Aggregate**
- Set “Fields to aggregate” to include:
  - `last_week_data`
  - `last_2_weeks_data`
- Connect: **Merge** → **Aggregate**
- This will produce arrays so the prompt can reference `[0]`.

9) **Add AI components**
- Add node: **OpenAI Chat Model (LangChain)**
  - Model: **gpt-5.1**
  - Attach **OpenAI API** credentials.
- Add node: **Calculator Tool (LangChain)**
- Add node: **AI Agent (LangChain)** named “AI Analyst”
  - Prompt: paste the analyst prompt text and keep the HTML-only requirement.
  - Ensure the interpolations point to:
    - `$json.last_week_data[0]....`
    - `$json.last_2_weeks_data[0]....`
  - Connect:
    - **Aggregate** → **AI Analyst** (main)
    - **OpenAI Chat Model** → **AI Analyst** (ai_languageModel)
    - **Calculator** → **AI Analyst** (ai_tool)

10) **Add Gmail send node**
- Add node: **Gmail** → operation “Send”
- Credentials: **Gmail OAuth2**
- To: your recipient(s)
- Subject: e.g., “Weekly Report”
- Body/message: set to the AI output field (in the template it is `{{$json.output}}`).
  - If your AI Agent outputs under a different key, map accordingly (verify by executing once).
- Connect: **AI Analyst** → **Gmail**

11) **Credentials checklist**
- Google Ads OAuth2 API credential:
  - Must have access to the specified Customer ID.
  - Developer token must be valid and permitted for the account.
- OpenAI API credential:
  - Must allow access to `gpt-5.1`.
- Gmail OAuth2:
  - Must allow sending email from the connected mailbox.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace `[Customer ID]` in both Google Ads HTTP Request URLs (format XXX-XXX-XXXX). | From Sticky Note5 |
| Add your Google Ads **Developer Token** to both HTTP Request header parameters. | From Sticky Note5 |
| Connect credentials: Google Ads OAuth2, Gmail OAuth2, OpenAI API. | From Sticky Note5 |
| Update recipient email in “Email Report to User”. | From Sticky Note5 |
| Runs every Monday at midnight; schedule is configurable. | From Sticky Note1 / Sticky Note5 |
| Intended comparison is “Last 7 days vs previous 7 days”; ensure the “previous” query matches that window. | From Sticky Note5 (and implied by workflow purpose) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.