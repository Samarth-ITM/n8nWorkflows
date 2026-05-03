Re-engage dormant leads with Claude emails using Crunchbase, NewsAPI, Hunter, and Gmail

https://n8nworkflows.xyz/workflows/re-engage-dormant-leads-with-claude-emails-using-crunchbase--newsapi--hunter--and-gmail-12712


# Re-engage dormant leads with Claude emails using Crunchbase, NewsAPI, Hunter, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Re-engage dormant leads with Claude emails using Crunchbase, NewsAPI, Hunter, and Gmail  
**Internal workflow name:** Re-engage dormant leads with AI emails and trigger detection  
**Status:** Inactive (disabled)  
**Purpose:** Identify CRM leads that have been inactive for 90+ days, detect “trigger events” (funding/news/leadership signals), generate a personalized re-engagement email using Claude (Anthropic), then notify the assigned sales rep via Gmail with a suggested draft.

### 1.1 Entry & Scheduling
Runs either manually (for testing) or automatically every week (Monday by default).

### 1.2 Lead Retrieval (CRM)
Loads leads (currently mocked) that represent dormant opportunities.

### 1.3 Dormancy Filtering (90+ days)
Filters only leads inactive for at least 90 days and enriches them with `days_inactive`.

### 1.4 Trigger Detection (parallel enrichment)
For each dormant lead, three checks run in parallel:
- Crunchbase autocomplete (proxy for funding signals)
- NewsAPI recent articles
- Hunter domain search (proxy for exec/leadership presence)

### 1.5 Trigger Analysis & Gating
Combines results, extracts trigger events, and **drops leads with no triggers** (workflow stops for those items).

### 1.6 AI Email Generation (Claude)
Uses an n8n LangChain Agent node with Anthropic as the chat model to draft a short re-engagement email.

### 1.7 Rep Notification via Gmail
Formats a rep-facing notification containing lead context, triggers, and the draft, then emails the assigned rep.

---

## 2. Block-by-Block Analysis

### Block A — Entry & Orchestration
**Overview:** Starts the workflow either weekly or manually for testing; both entry nodes converge into lead loading.  
**Nodes involved:**  
- Weekly schedule trigger  
- Test workflow manually  

#### Node: **Weekly schedule trigger**
- **Type / role:** Schedule Trigger — time-based workflow entry.
- **Configuration (interpreted):** Runs weekly on **Monday** (`triggerAtDay: [1]`).
- **Outputs:** Sends execution to **Load inactive leads (mock)**.
- **Edge cases / failures:**
  - Timezone differences based on n8n instance settings may shift “Monday” boundaries.
  - If the workflow is inactive, schedule will not run.

#### Node: **Test workflow manually**
- **Type / role:** Manual Trigger — interactive entry for development/testing.
- **Configuration:** Default manual trigger.
- **Outputs:** Sends execution to **Load inactive leads (mock)**.
- **Edge cases:** None; only runs when manually executed.

---

### Block B — CRM Lead Loading (Mock) and Replacement Point
**Overview:** Produces a list of lead records (currently hard-coded) that represent previously contacted leads. Intended to be replaced by a real CRM query for “inactive for 90+ days.”  
**Nodes involved:**  
- Load inactive leads (mock)

#### Node: **Load inactive leads (mock)**
- **Type / role:** Code node — generates lead items.
- **Configuration choices:**
  - Returns an array of lead objects with fields used downstream:  
    `id, company, contact_name, contact_email, title, status, last_activity_date, industry, company_size, assigned_rep, original_opportunity_value, loss_reason`
  - Converts each lead into an n8n item: `{ json: lead }`.
- **Outputs:** To **Filter dormant leads (90+ days)**.
- **Edge cases / failures:**
  - `last_activity_date` must be parseable by `new Date(...)`. Non-ISO formats may parse inconsistently across environments.
  - Uses `user@example.com` placeholders; real sending will go to these addresses unless replaced.
- **Sub-workflow reference:** None.

---

### Block C — Dormancy Filter (90+ days)
**Overview:** Filters to only items whose `last_activity_date` is older than 90 days, and adds `days_inactive`.  
**Nodes involved:**  
- Filter dormant leads (90+ days)

#### Node: **Filter dormant leads (90+ days)**
- **Type / role:** Code node — filtering + enrichment.
- **Configuration choices:**
  - Computes `ninetyDaysAgo = today - 90 days`.
  - For each input item:
    - If `new Date(last_activity_date) < ninetyDaysAgo`, it keeps it and adds:
      - `days_inactive = floor((today - lastActivity)/day)`
- **Outputs:** Fans out in parallel to:
  - Check funding events (Crunchbase)
  - Check company news (NewsAPI)
  - Check leadership changes (Hunter)
- **Edge cases / failures:**
  - Invalid date parsing results in `Invalid Date`, comparisons may fail silently and drop leads unexpectedly.
  - Time-of-day and timezone can slightly affect boundary cases (exactly 90 days).

---

### Block D — Trigger Detection (Parallel API Requests)
**Overview:** Enriches each dormant lead with external signals. All three HTTP requests run in parallel for each lead item and continue even if any request fails.  
**Nodes involved:**  
- Check funding events (Crunchbase)  
- Check company news (NewsAPI)  
- Check leadership changes (Hunter)

#### Node: **Check funding events (Crunchbase)**
- **Type / role:** HTTP Request — queries Crunchbase autocomplete.
- **Configuration choices:**
  - URL: `https://api.crunchbase.com/api/v4/autocompletes?query={{ $json.company }}`
  - Auth: Generic credential type using **HTTP Header Auth**
  - Header: `X-cb-user-key: {{ $credentials.crunchbaseApiKey }}`
  - Requests **fullResponse** (captures headers/status/body).
  - **continueOnFail: true** to avoid stopping the lead if Crunchbase errors.
- **Inputs:** From **Filter dormant leads (90+ days)**.
- **Outputs:** To **Analyze trigger events**.
- **Edge cases / failures:**
  - Missing/invalid API key → 401/403; execution continues but data may be error-shaped.
  - Autocomplete is not a guaranteed funding endpoint; may not include `last_funding_type` fields depending on API response format/plan.
  - Because fullResponse is enabled, downstream code expecting a simple JSON body may not match (depends on n8n’s returned structure).

#### Node: **Check company news (NewsAPI)**
- **Type / role:** HTTP Request — fetches recent articles.
- **Configuration choices:**
  - URL: `https://newsapi.org/v2/everything?q={{ encodeURIComponent($json.company) }}&sortBy=publishedAt&pageSize=5`
  - Auth: Generic credential type using **Query Auth**:
    - `apiKey={{ $credentials.newsApiKey }}`
  - **continueOnFail: true**
- **Inputs:** From **Filter dormant leads (90+ days)**.
- **Outputs:** To **Analyze trigger events**.
- **Edge cases / failures:**
  - NewsAPI free tier restrictions (rate limits, limited sources) can cause 429 or empty results.
  - If response is an error JSON, downstream `articles` checks may fail gracefully (code checks for existence).

#### Node: **Check leadership changes (Hunter)**
- **Type / role:** HTTP Request — domain search (proxy for leadership/executive contacts).
- **Configuration choices:**
  - URL: `https://api.hunter.io/v2/domain-search?domain={{ $json.company.toLowerCase().replace(/[^a-z0-9]/g, '') }}.com`
    - Builds a guessed domain by stripping non-alphanumerics and appending `.com`.
  - Auth: Generic credential type using **Query Auth**:
    - `api_key={{ $credentials.hunterApiKey }}`
  - **continueOnFail: true**
- **Inputs:** From **Filter dormant leads (90+ days)**.
- **Outputs:** To **Analyze trigger events**.
- **Edge cases / failures:**
  - Domain guessing is fragile (e.g., “CloudFirst Inc” → `cloudfirstinc.com` might be wrong).
  - Hunter may return no `data.emails` for many companies, even if domain is correct.
  - Rate limiting / quota errors possible.

---

### Block E — Trigger Analysis & Lead Gating
**Overview:** Merges the lead + all enrichment results, detects qualifying triggers, and drops the lead if no triggers were found (prevents unnecessary AI/email sending).  
**Nodes involved:**  
- Analyze trigger events

#### Node: **Analyze trigger events**
- **Type / role:** Code node — aggregation + business rules.
- **Configuration choices (logic):**
  - Reads:
    - Lead: `$('Filter dormant leads (90+ days)').item.json`
    - Crunchbase: `$('Check funding events (Crunchbase)').item.json || {}`
    - NewsAPI: `$('Check company news (NewsAPI)').item.json || {}`
    - Hunter: `$('Check leadership changes (Hunter)').item.json || {}`
  - Builds `triggerEvents[]` from:
    - **Crunchbase:** if `entities[0].properties.last_funding_type` exists → adds Funding trigger.
    - **NewsAPI:** if `articles[0].publishedAt` is within last 30 days → adds News trigger.
    - **Hunter:** filters `data.emails[]` for senior roles (vp/chief/director/head) → adds Leadership trigger (presence-based).
  - If no triggers: `return [];` (this item stops here).
  - Otherwise output includes:
    - `trigger_events: [...]`
    - `primary_trigger: triggerEvents[0]` (first trigger wins; ordering is Crunchbase then News then Hunter).
- **Inputs:** Receives three incoming connections (Crunchbase/NewsAPI/Hunter). Also references the dormancy-filter node via selector.
- **Outputs:** To **Generate re-engagement email**.
- **Edge cases / failures:**
  - If HTTP nodes return “full response” wrappers or error objects, expected fields like `entities` / `articles` may not exist; triggers may never be detected.
  - Multiple inputs: the node will execute when it receives items from upstream connections; using `$('Node').item` assumes item alignment. If one branch errors or returns different item counts, cross-node item selection can mismatch.
  - Primary trigger selection is simplistic; may choose a less relevant trigger if present first.

---

### Block F — AI Email Generation (Claude via Anthropic)
**Overview:** Uses Claude to draft a short, formal, personalized re-engagement email that references the detected trigger and prior history.  
**Nodes involved:**  
- Generate re-engagement email  
- Claude Sonnet 4 (Anthropic chat model)

#### Node: **Generate re-engagement email**
- **Type / role:** LangChain Agent node — generates text using an attached language model.
- **Configuration choices:**
  - Prompt is built via expressions from the lead item:
    - Contact fields: `contact_name`, `title`, `company`, `industry`, `company_size`
    - History: `status`, `days_inactive`, `loss_reason`
    - Trigger: `primary_trigger.type/details/source`
  - **System message** enforces:
    - Professional/formal, concise (<150 words)
    - No subject line in body
    - End with `[Your Name]`
    - Avoid “I hope this email finds you well”
- **Model dependency / connections:**
  - Uses an AI language model input from **Claude Sonnet 4** via `ai_languageModel` connection.
- **Outputs:** To **Format notification for rep**.
- **Edge cases / failures:**
  - Model may occasionally violate constraints (word count, inclusion of subject, sign-off). No validation node enforces compliance.
  - If trigger data is missing (e.g., primary_trigger undefined), prompt expressions can produce empty strings or fail in some cases.

#### Node: **Claude Sonnet 4**
- **Type / role:** Anthropic Chat Model node — provides the LLM for the agent.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929` (displayed as “Claude Sonnet 4.5”)
  - Temperature: `0.7`
  - Max tokens: `1024`
- **Credentials:** Anthropic API credential (“Anthropic account”).
- **Outputs:** Feeds **Generate re-engagement email** as its language model.
- **Edge cases / failures:**
  - Credential misconfiguration → authentication failures.
  - Model availability/name changes can break execution if the model is deprecated.

---

### Block G — Rep Notification Formatting & Gmail Send
**Overview:** Creates a rep-friendly summary (lead details, triggers, and AI draft), then emails it to the assigned rep via Gmail.  
**Nodes involved:**  
- Format notification for rep  
- Send notification (Gmail)

#### Node: **Format notification for rep**
- **Type / role:** Code node — normalization + message composition.
- **Configuration choices:**
  - Reads:
    - Lead: `$('Analyze trigger events').item.json`
    - AI output: `$('Generate re-engagement email').item.json`
  - Extracts draft from multiple possible agent output shapes:
    - `aiResponse.output || aiResponse.text || aiResponse.message?.content || aiResponse.content || 'Error: Could not generate email'`
  - Builds:
    - `to = lead.assigned_rep`
    - `subject = Re-engagement Opportunity: <company> - <trigger type> Trigger`
    - `body` includes a formatted section plus:
      - A **draft subject**: `Congratulations on ${lead.primary_trigger.details}`
      - The **draft body** from Claude
    - Also outputs `lead_id`, `lead_email`, `draft_subject`, `draft_body`
- **Outputs:** To **Send notification (Gmail)**.
- **Edge cases / failures:**
  - `original_opportunity_value.toLocaleString()` will throw if value is missing or not a number.
  - If `trigger_events` is missing/empty, `.map` will throw; however upstream code should ensure triggers exist.
  - AI output may include markdown or extra whitespace; no sanitization is applied.

#### Node: **Send notification (Gmail)**
- **Type / role:** Gmail node — sends email to the rep.
- **Configuration choices:**
  - To: `{{ $json.to }}`
  - Subject: `{{ $json.subject }}`
  - Message: `{{ $json.body }}`
- **Credentials:** Requires Gmail OAuth2 credentials configured in n8n for the sending account.
- **Edge cases / failures:**
  - OAuth token expiration / missing scopes → auth errors.
  - Sending limits, “from” alias restrictions, or spam policies can block delivery.
  - If `to` is empty/invalid, node will fail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly schedule trigger | Schedule Trigger | Weekly automation entry (Monday) | — | Load inactive leads (mock) | ## How it works / Setup steps (workflow description and setup checklist) |
| Test workflow manually | Manual Trigger | Manual entry for testing | — | Load inactive leads (mock) | ## How it works / Setup steps (workflow description and setup checklist) |
| Load inactive leads (mock) | Code | Mock CRM lead source (to be replaced) | Weekly schedule trigger; Test workflow manually | Filter dormant leads (90+ days) | **CRM Data** Replace mock data with your CRM (Salesforce, HubSpot, Pipedrive) |
| Filter dormant leads (90+ days) | Code | Filter and enrich leads by inactivity | Load inactive leads (mock) | Check funding events (Crunchbase); Check company news (NewsAPI); Check leadership changes (Hunter) | **CRM Data** Replace mock data with your CRM (Salesforce, HubSpot, Pipedrive) |
| Check funding events (Crunchbase) | HTTP Request | Trigger detection: funding signal | Filter dormant leads (90+ days) | Analyze trigger events | **Trigger Detection** Checks funding, news, and leadership changes in parallel |
| Check company news (NewsAPI) | HTTP Request | Trigger detection: recent news | Filter dormant leads (90+ days) | Analyze trigger events | **Trigger Detection** Checks funding, news, and leadership changes in parallel |
| Check leadership changes (Hunter) | HTTP Request | Trigger detection: exec/leadership presence | Filter dormant leads (90+ days) | Analyze trigger events | **Trigger Detection** Checks funding, news, and leadership changes in parallel |
| Analyze trigger events | Code | Aggregate sources; gate leads with triggers | Check funding events (Crunchbase); Check company news (NewsAPI); Check leadership changes (Hunter) | Generate re-engagement email | **Trigger Detection** Checks funding, news, and leadership changes in parallel |
| Claude Sonnet 4 | Anthropic Chat Model (LangChain) | LLM provider for Claude | — | Generate re-engagement email (ai_languageModel) | **AI Generation** Claude writes personalized re-engagement emails based on triggers |
| Generate re-engagement email | LangChain Agent | Draft personalized re-engagement email | Analyze trigger events; Claude Sonnet 4 (model input) | Format notification for rep | **AI Generation** Claude writes personalized re-engagement emails based on triggers |
| Format notification for rep | Code | Build rep notification + embed draft | Generate re-engagement email | Send notification (Gmail) | **Email Output** Formats and sends draft to sales rep for approval |
| Send notification (Gmail) | Gmail | Send rep notification email | Format notification for rep | — | **Email Output** Formats and sends draft to sales rep for approval |
| Sticky Note | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note1 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note2 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note3 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note4 | Sticky Note | Comment / documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Re-engage dormant leads with AI emails and trigger detection*
   - (Optional) Add tags: `sales-automation`, `lead-reengagement`

2. **Add entry nodes**
   1) Add **Schedule Trigger**
   - Set interval: **Weeks**
   - Set “trigger at day”: **Monday**
   - Name: *Weekly schedule trigger*
   
   2) Add **Manual Trigger**
   - Name: *Test workflow manually*

3. **Add lead source node (mock or CRM)**
   - Add **Code** node
   - Name: *Load inactive leads (mock)*
   - Paste logic that outputs one item per lead (each item must have at least):
     - `company`, `contact_name`, `contact_email`, `title`
     - `last_activity_date`, `assigned_rep`
     - plus any optional business fields used later
   - Connect:
     - **Weekly schedule trigger → Load inactive leads (mock)**
     - **Test workflow manually → Load inactive leads (mock)**

   **If replacing with a real CRM integration:**
   - Replace the Code node with the appropriate CRM node (Salesforce/HubSpot/Pipedrive) or an **HTTP Request** node.
   - Ensure the output fields match what downstream nodes expect (or add a mapping Code/Set node).

4. **Add dormancy filter**
   - Add **Code** node
   - Name: *Filter dormant leads (90+ days)*
   - Implement:
     - Parse `last_activity_date`
     - Keep only leads older than 90 days
     - Add `days_inactive`
   - Connect: **Load inactive leads (mock) → Filter dormant leads (90+ days)**

5. **Add trigger detection HTTP nodes (3 in parallel)**
   1) **Crunchbase**
   - Add **HTTP Request**
   - Name: *Check funding events (Crunchbase)*
   - Method: GET
   - URL expression: `https://api.crunchbase.com/api/v4/autocompletes?query={{ $json.company }}`
   - Authentication: **Generic Credential Type**
     - Type: **HTTP Header Auth**
     - Header: `X-cb-user-key` = `{{ $credentials.crunchbaseApiKey }}`
   - Enable **Continue On Fail**
   - (Optional) Enable “Full response” only if your parsing logic expects it; otherwise keep default body-only.

   2) **NewsAPI**
   - Add **HTTP Request**
   - Name: *Check company news (NewsAPI)*
   - Method: GET
   - URL expression: `https://newsapi.org/v2/everything?q={{ encodeURIComponent($json.company) }}&sortBy=publishedAt&pageSize=5`
   - Authentication: **Generic Credential Type**
     - Type: **HTTP Query Auth**
     - Query param: `apiKey` = `{{ $credentials.newsApiKey }}`
   - Enable **Continue On Fail**

   3) **Hunter**
   - Add **HTTP Request**
   - Name: *Check leadership changes (Hunter)*
   - Method: GET
   - URL expression: `https://api.hunter.io/v2/domain-search?domain={{ $json.company.toLowerCase().replace(/[^a-z0-9]/g, '') }}.com`
   - Authentication: **Generic Credential Type**
     - Type: **HTTP Query Auth**
     - Query param: `api_key` = `{{ $credentials.hunterApiKey }}`
   - Enable **Continue On Fail**

   - Connect fan-out:
     - **Filter dormant leads (90+ days) →** each of the three HTTP nodes.

6. **Add trigger analysis / gating**
   - Add **Code** node
   - Name: *Analyze trigger events*
   - Implement:
     - Pull JSON from each HTTP node + base lead
     - Build `trigger_events[]`
     - If none found: `return []`
     - Else return lead enriched with `trigger_events` + `primary_trigger`
   - Connect:
     - **Check funding events (Crunchbase) → Analyze trigger events**
     - **Check company news (NewsAPI) → Analyze trigger events**
     - **Check leadership changes (Hunter) → Analyze trigger events**

7. **Add Claude model node (Anthropic)**
   - Add **Anthropic Chat Model** node (LangChain)
   - Name: *Claude Sonnet 4*
   - Select model: `claude-sonnet-4-5-20250929` (or closest available Sonnet model in your instance)
   - Set:
     - Temperature: `0.7`
     - Max tokens: `1024`
   - Configure **Anthropic API credentials** in n8n (API key).

8. **Add AI generation agent**
   - Add **Agent** node (LangChain)
   - Name: *Generate re-engagement email*
   - Set prompt text using lead + trigger fields (as in the workflow):
     - Include lead info, history, and detected primary trigger
   - Set **System Message** with constraints (formal, <150 words, end with `[Your Name]`, etc.)
   - Connect:
     - **Analyze trigger events → Generate re-engagement email**
     - **Claude Sonnet 4 → Generate re-engagement email** via the node’s **AI Language Model** connection.

9. **Add formatting node for rep notification**
   - Add **Code** node
   - Name: *Format notification for rep*
   - Build:
     - `to`, `subject`, `body`, `draft_subject`, `draft_body`
   - Include robust extraction of agent output (e.g., `output/text/message.content` fallback).
   - Connect: **Generate re-engagement email → Format notification for rep**

10. **Add Gmail send node**
   - Add **Gmail** node
   - Name: *Send notification (Gmail)*
   - Operation: Send
   - Map:
     - To: `{{ $json.to }}`
     - Subject: `{{ $json.subject }}`
     - Message: `{{ $json.body }}`
   - Configure **Gmail OAuth2 credentials** in n8n (scopes must allow sending).
   - Connect: **Format notification for rep → Send notification (Gmail)**

11. **Test and enable**
   - Run via **Test workflow manually** with a real email address for `assigned_rep`.
   - Confirm triggers are detected and Gmail sends successfully.
   - Enable the workflow to activate the weekly schedule.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| NewsAPI credentials required (free tier available). | https://newsapi.org |
| Replace “Load inactive leads (mock)” with your CRM integration (Salesforce/HubSpot/etc.). | Workflow setup note (CRM replacement point) |
| Add credentials: NewsAPI, Crunchbase (optional/paid), Hunter.io, Anthropic API, Gmail OAuth. | Workflow setup note |
| Trigger detection runs in parallel; only leads with triggers proceed to AI + email. | Workflow behavior note |
| Use manual trigger to test before enabling schedule. | Workflow setup note |