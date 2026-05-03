Enrich B2B leads for Attio CRM with Apollo, LinkedIn, news and GPT-4o

https://n8nworkflows.xyz/workflows/enrich-b2b-leads-for-attio-crm-with-apollo--linkedin--news-and-gpt-4o-12710


# Enrich B2B leads for Attio CRM with Apollo, LinkedIn, news and GPT-4o

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *B2B Lead Enrichment - Attio CRM*  
**Purpose:** Enrich B2B company leads in **Attio CRM** by combining **Apollo company data**, **latest news**, **LinkedIn company activity**, **LinkedIn leadership activity**, and **GPT‑4o** summarization + verification, then writing structured enrichment fields back to Attio.

### 1.1 Goals and success criteria (as stated in sticky notes)
- **Business outcome:** Reduce BDR time spent on lead research; reinvest time into relationship building.
- **System outcomes:** Output trustworthiness, minimal chance of awkward/incorrect claims, “human-equivalent” quick enrichment, fallbacks/graceful handling when sources are unavailable.
- **Metrics:**  
  - Direct: avg time BDR spent enriching leads  
  - Proxy/lagging: opportunities won delta, time cultivating leads, deal size delta

### 1.2 Logical blocks (based on dependencies)
1. **Input & Apollo company enrichment** → validate response → normalize fields.
2. **Attio company upsert (baseline record)** → create/update company by domain.
3. **Parallel enrichment streams** (fan-out from company record):
   - Latest news → content extraction (Tavily) → LLM summary agent
   - LinkedIn company page posts → last 5 posts → LLM summary agent
   - Leadership discovery (LLM with web search) → per-person Apollo match → LinkedIn profile scrape → LLM leadership-summary agent → optional Attio person upsert
4. **Merge + LLM-as-judge verification** of the three summaries.
5. **Final enrichment agent** produces structured dossier → format → write back to Attio company record.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Apollo Company Enrichment
**Overview:** Starts the workflow manually, calls Apollo to enrich a company by domain, checks that enrichment returned an `organization`, and maps key organization fields into a flatter structure used throughout the workflow.

**Nodes involved:**
- When clicking 'Execute workflow'
- Apollo:EnrichCompany
- If1
- SetFields

#### Node: **When clicking 'Execute workflow'**
- **Type/role:** Manual Trigger; entry point for testing.
- **Configuration:** No parameters.
- **Outputs:** One item to **Apollo:EnrichCompany**.
- **Edge cases:** None (manual execution only).

#### Node: **Apollo:EnrichCompany**
- **Type/role:** HTTP Request; Apollo Organizations Enrich endpoint.
- **Configuration (interpreted):**
  - `GET` (default) to `https://api.apollo.io/api/v1/organizations/enrich`
  - Sends query parameter `domain={{$json.domain}}`
  - Auth: Header auth credential (“Apollo API”).
- **Inputs:** From manual trigger; expects upstream JSON to include `domain`.
- **Outputs:** Apollo response; expected field `organization`.
- **Edge cases/failures:**
  - Missing `$json.domain` → Apollo returns error/empty.
  - 401/403 if Apollo key invalid; 429 rate limiting; network timeouts.
  - Apollo may return no `organization` for unknown domains.

#### Node: **If1**
- **Type/role:** IF; gate the workflow if Apollo returned an organization.
- **Condition:** Checks existence of `{{$json.organization}}`.
- **True path:** Goes to **SetFields**.
- **False path:** Unconnected (workflow effectively stops).
- **Edge cases:**
  - Apollo may return `organization: null`; condition fails (expected).
  - If Apollo response shape changes, expression may mis-evaluate.

#### Node: **SetFields**
- **Type/role:** Set; normalize and flatten Apollo `organization` object.
- **Key mappings (examples):**
  - `id={{$json.organization.id}}`
  - `name={{$json.organization.name}}`
  - `linkedin_url={{$json.organization.linkedin_url.replace('http','https')}}`
  - `primary_domain={{$json.organization.primary_domain}}`
  - plus phones, social URLs, size, address, description, etc.
- **Outputs:** To **Post Assert Body: Company**.
- **Edge cases:**
  - `.replace('http','https')` will throw if `linkedin_url` is `null/undefined`. (No guard is present.)
  - Some mapped fields are typed as `number/object/array`; malformed Apollo data can cause type mismatches in strict contexts.

---

### Block 2 — Attio Company Upsert (Baseline)
**Overview:** Formats a minimal Attio “company” record payload (name, description, LinkedIn, domain) and upserts it into Attio using domains as the matching attribute. This record becomes the anchor for later enrichment.

**Nodes involved:**
- Post Assert Body: Company
- Assert Record: Company

#### Node: **Post Assert Body: Company**
- **Type/role:** Code (JavaScript); builds Attio v2 record payload.
- **Configuration choices:**
  - Always sets:
    - `values.name = [org.name || '']`
    - `values.description = [org.short_description || '']`
    - `values.linkedin = [org.linkedin_url || '']`
  - Conditionally sets `values.domains = [{domain: org.primary_domain}]` if `primary_domain` not empty.
- **Outputs:** `{ data: { values } }` to **Assert Record: Company**.
- **Edge cases:**
  - Writes empty strings into Attio arrays if name/description/linkedin missing; may be undesirable depending on Attio field validation.
  - If `primary_domain` missing, record match via domains might not work, creating duplicates if other matching is needed.

#### Node: **Assert Record: Company**
- **Type/role:** Attio node; upsert company record.
- **Configuration:**
  - Object: `companies`
  - Operation: `PUT -v2-objects--object--records`
  - Matching attribute: `domains`
  - Data: `{{$json.data}}`
- **Outputs:** Fans out into three enrichment streams:
  - **Apollo:LatestNews**
  - **Leadership Search Agent**
  - **Get a linked in company page**
- **Failure modes:**
  - Attio auth misconfigured (401/403).
  - Attio schema mismatch (field names like `linkedin`, `domains`, `description` must exist).
  - If `domains` omitted, matching may fail (possible duplicate record creation or API error depending on Attio behavior).

---

### Block 3 — Latest News Enrichment (Apollo → Tavily Extract → LLM Summary)
**Overview:** Pulls recent news articles from Apollo for the company, takes up to 5, extracts page content via Tavily, aggregates it, and asks an LLM agent to summarize into CRM-relevant talking points.

**Nodes involved:**
- Apollo:LatestNews
- Split News
- Last 5 Articles
- Extract (Tavily)
- Edit Fields2
- Aggregate2
- LLMHelper
- Apollo Summary Agent
- OpenAI Chat Model

#### Node: **Apollo:LatestNews**
- **Type/role:** HTTP Request; Apollo news search.
- **Configuration:**
  - `POST https://api.apollo.io/api/v1/news_articles/search`
  - Query parameters:
    - `organization_ids[]={{$('SetFields').item.json.id}}`
    - `published_at[min]={{$now.minus(6,'months')}}`
  - Auth: Apollo header auth
- **Outputs:** Expected field: `news_articles` array.
- **Edge cases:**
  - If `SetFields` didn’t produce `id`, request may return empty.
  - Time formatting: `$now.minus(...)` yields a DateTime; Apollo may require a specific ISO format.

#### Node: **Split News**
- **Type/role:** Split Out; iterates `news_articles`.
- **Field:** `news_articles`
- **Outputs:** One item per article to **Last 5 Articles**.
- **Edge cases:** If `news_articles` missing/not array → node errors.

#### Node: **Last 5 Articles**
- **Type/role:** Limit; cap to 5 items.
- **Outputs:** To **Extract**.

#### Node: **Extract** (Tavily)
- **Type/role:** Tavily Extract; fetch and extract raw content from URLs.
- **Configuration:**
  - Resource: `extract`
  - `urls: [{{$json.url}}]`
  - `onError: continueRegularOutput` (important graceful degradation)
- **Outputs:** Tavily result includes `results[0].raw_content`.
- **Edge cases:**
  - Missing/invalid `$json.url`
  - Paywalls/blocked sites → empty content
  - Tavily quota/rate limits; node continues (but downstream may get empty text)

#### Node: **Edit Fields2**
- **Type/role:** Set; keep only extracted raw content.
- **Field:** `raw_extract={{$json.results[0].raw_content}}`
- **Edge cases:** If `results[0]` missing → expression error.

#### Node: **Aggregate2**
- **Type/role:** Aggregate all item data into a single item (list).
- **Purpose:** Prepare combined corpus for LLM prompt.
- **Outputs:** To **LLMHelper**.

#### Node: **LLMHelper**
- **Type/role:** Code; converts aggregated JSON into a single prompt string.
- **Behavior:**
  - Collects `$input.all().map(item => item.json)`
  - `JSON.stringify(..., null, 2)` and wraps in a prompt “Please analyze this data.”
- **Outputs:** `{ prompt: llmPrompt }` to **Apollo Summary Agent**.
- **Edge cases:** Large extracted content can exceed model context window.

#### Node: **Apollo Summary Agent**
- **Type/role:** LangChain Agent; summarizes news-derived info.
- **Inputs:** `text={{$json.prompt}}`
- **System message highlights:**
  - Role: data extraction agent for CRM enrichment
  - Goal: key information about the target org via recent article activity; convert to talking points
  - Guardrail: must only provide info relevant to the org
- **Model connection:** Uses **OpenAI Chat Model** as `ai_languageModel`.
- **Outputs:** Sent to **Merge** (input index 0).
- **Edge cases:**
  - If prompt is empty/too long, output quality degrades.
  - Not using a structured parser here → freeform output may be inconsistent.

#### Node: **OpenAI Chat Model**
- **Type/role:** OpenAI chat model provider for the agent.
- **Model:** `gpt-4o`
- **Edge cases:** API auth, rate limits, token limits.

---

### Block 4 — LinkedIn Company Enrichment (Scrape Creators → Posts → LLM Summary)
**Overview:** Scrapes the LinkedIn company page, extracts posts, limits to last 5, aggregates post text, and summarizes into CRM talking points.

**Nodes involved:**
- Get a linked in company page
- Split Out1
- Last 5 Posts
- Extract Post Text
- Aggregate
- LLMHelper2
- LinkedIn Company Summary Agent
- OpenAI Chat Model2

#### Node: **Get a linked in company page**
- **Type/role:** Scrape Creators (LinkedIn) operation: get company page.
- **Configuration:**
  - `url={{$('SetFields').item.json.linkedin_url}}`
  - Resource: `linkedin`, Operation: `getLinkedinCompanyPage`
- **Outputs:** Expected `posts` array for next split.
- **Edge cases:**
  - LinkedIn URL missing/invalid; scraping provider blocks; throttling.
  - If SetFields `.replace()` previously failed, this never runs.

#### Node: **Split Out1**
- **Type/role:** Split Out; splits `posts`.
- **Outputs:** One post per item to **Last 5 Posts**.

#### Node: **Last 5 Posts**
- **Type/role:** Limit to 5 items.

#### Node: **Extract Post Text**
- **Type/role:** Set; keep `text={{$json.text}}`.

#### Node: **Aggregate**
- **Type/role:** Aggregate all post texts into one combined dataset.

#### Node: **LLMHelper2**
- **Type/role:** Code; formats aggregated posts JSON into a prompt.

#### Node: **LinkedIn Company Summary Agent**
- **Type/role:** LangChain Agent; summarizes company post activity into talking points.
- **System message highlights:** LinkedIn extraction agent; only info relevant to org.
- **Model connection:** **OpenAI Chat Model2**
- **Outputs:** To **Merge** (input index 1).

#### Node: **OpenAI Chat Model2**
- **Type/role:** OpenAI model provider (`gpt-4o`).

---

### Block 5 — Leadership Enrichment (LLM discovery → Apollo match → LinkedIn scrape → LLM summary → optional Attio person upsert)
**Overview:** Uses an LLM (with web search capability enabled in the model node) to produce a *structured leadership team list*, then for each leader tries to match a person in Apollo, normalizes LinkedIn URL, optionally scrapes the LinkedIn profile, summarizes leadership activity, and (if email + domain match) upserts the person into Attio.

**Nodes involved:**
- Leadership Search Agent
- GPT 4o - Date Smart
- Structured Output Parser
- Split People
- Apollo:Fetch Person
- Edit Fields
- If LinkedIn Url
- Get a LinkedIn Profile
- Edit Fields1
- Aggregate1
- LLMHelper1
- LinkedIn Leadership Summary
- OpenAI Chat Model1
- Edit Fields (person linkedin normalize)
- Set Person Fields
- If Email & Domain
- Post Assert Body
- Assert a record

#### Node: **Leadership Search Agent**
- **Type/role:** LangChain Agent; finds leadership team and outputs structured JSON.
- **Input text:**
  - Company Name from `SetFields`
  - Domain from Apollo enrich: `{{$('Apollo:EnrichCompany').item.json.organization.primary_domain}}`
- **System message highlights:**
  - Must be current as of `$now`
  - Must only include currently employed leaders
- **Output parsing:** `hasOutputParser: true` connected to **Structured Output Parser**.
- **Model connection:** **GPT 4o - Date Smart**
- **Outputs:** To **Split People**
- **Edge cases:**
  - Hallucinated leaders; mitigated partially by later “judge” step but still risk.
  - Parser failures if agent output doesn’t conform to schema.

#### Node: **Structured Output Parser**
- **Type/role:** Structured parser; schema example:
  ```json
  { "leadership_team": [ { "name": "...", "role": "...", "asof": "Dec 2024" } ] }
  ```
- **Role:** Forces Leadership Search Agent output into the expected structure.

#### Node: **GPT 4o - Date Smart**
- **Type/role:** OpenAI chat model provider (`gpt-4o`) with **built-in webSearch tool enabled**.
- **Why it matters:** Improves “as of now” leadership correctness, but may increase latency/cost and variability.

#### Node: **Split People**
- **Type/role:** Split Out; field `output.leadership_team`.
- **Outputs:** One leader per item to **Apollo:Fetch Person**.

#### Node: **Apollo:Fetch Person**
- **Type/role:** HTTP Request; Apollo people match.
- **Configuration:**
  - `POST https://api.apollo.io/api/v1/people/match`
  - Query parameters:
    - `name={{$json.name}}` (leader name)
    - `organization_name={{$('Apollo:EnrichCompany').item.json.organization.name}}`
- **Outputs:** Two parallel branches:
  - to **Edit Fields** (LinkedIn URL normalization)
  - to **Set Person Fields** (Attio person upsert preparation)
- **Edge cases:**
  - Apollo may return ambiguous/no matches; `person` may be missing.
  - Rate limits.

#### Node: **Edit Fields** (leadership branch)
- **Type/role:** Set; normalize person LinkedIn URL:
  - `person.linkedin_url={{$json.person.linkedin_url.replace('http','https')}}`
- **Outputs:** To **If LinkedIn Url**.
- **Edge cases:** If `person` or `person.linkedin_url` missing → expression error (no guard).

#### Node: **If LinkedIn Url**
- **Type/role:** IF; checks `person.linkedin_url` is not empty.
- **True:** **Get a LinkedIn Profile**
- **False:** no connection (leadership-summary path stops for that person).

#### Node: **Get a LinkedIn Profile**
- **Type/role:** Scrape Creators LinkedIn profile scrape.
- **Config:** `onError: continueRegularOutput` (graceful)
- **URL:** `{{$json.person.linkedin_url}}`
- **Outputs:** To **Edit Fields1**
- **Edge cases:** scraping blocks; profile private; throttling; continues with partial data.

#### Node: **Edit Fields1**
- **Type/role:** Set; keep relevant scraped profile fields:
  - `name`, `image`, `about`, `activity`, `publications`
- **Outputs:** **Aggregate1**

#### Node: **Aggregate1**
- **Type/role:** Aggregate all leadership profiles data (across leaders) into one payload.

#### Node: **LLMHelper1**
- **Type/role:** Code; converts aggregated leadership profile JSON to a prompt.

#### Node: **LinkedIn Leadership Summary**
- **Type/role:** LangChain Agent; summarizes leadership/employee activity.
- **Model connection:** **OpenAI Chat Model1**
- **Outputs:** To **Merge** (input index 2).

#### Node: **OpenAI Chat Model1**
- **Type/role:** OpenAI model provider (`gpt-4o`).

##### Person upsert sub-branch (Attio People)
This branch runs in parallel per leader match result from Apollo.

#### Node: **Set Person Fields**
- **Type/role:** Set; prepares person fields and carries company domain.
- **Key fields:**
  - `name` and `role` taken from `Split People` item
  - `person.linkedin_url`, `person.email_status`, `person.email` from Apollo match response
  - `data.values.domains[0].domain` pulled from **Post Assert Body: Company** output
- **Outputs:** To **If Email & Domain**
- **Edge cases:**
  - Cross-item reference `$('Post Assert Body: Company').item...` assumes correct item alignment; can break with multiple items or parallelism.
  - If `person` missing from Apollo match, many values become `undefined`.

#### Node: **If Email & Domain**
- **Type/role:** IF; only upsert people when:
  - `person.email` is not empty
  - email domain equals company domain:
    - `person.email.split("@")[1] == data.values.domains[0].domain`
- **True:** **Post Assert Body**
- **Edge cases:**
  - `split("@")[1]` fails if email malformed.
  - Company domain may be absent; comparison fails.

#### Node: **Post Assert Body**
- **Type/role:** Code; formats Attio People record payload.
- **Behavior:**
  - `values.name=[data.name]` (if present)
  - `values.job_title=[data.role]` (if present)
  - `values.linkedin=[https normalized]` only if includes `linkedin.com`
  - `values.email_addresses=[{email_address: trimmedEmail}]` only if non-empty
- **Outputs:** To **Assert a record**

#### Node: **Assert a record**
- **Type/role:** Attio upsert record (note: object not specified in node config).
- **Configuration:**
  - Resource: Records
  - Operation: PUT v2 objects/object/records
  - Matching attribute: `email_addresses`
- **Risk/edge case:** Since `object` is not set in this node (unlike the company node), Attio may error or default unexpectedly depending on node implementation/version. In most Attio use, you must specify object (e.g., `people`). Verify node config in UI.

---

### Block 6 — Merge Summaries + Judge Verification
**Overview:** Merges outputs from the three summarizers (news, LinkedIn company, LinkedIn leadership), concatenates them, then runs an LLM “judge” to verify claims and recommend updates.

**Nodes involved:**
- Merge
- Aggregate3
- Edit Fields3
- LLM Critic
- Think1
- OpenAI Chat Model3
- Structured Output Parser1

#### Node: **Merge**
- **Type/role:** Merge (3 inputs).
- **Configuration:** `numberInputs=3`
- **Inputs:**
  - Input 0: Apollo Summary Agent (news)
  - Input 1: LinkedIn Company Summary Agent (company posts)
  - Input 2: LinkedIn Leadership Summary (leadership activity)
- **Output:** To **Aggregate3**
- **Edge cases:** If one branch produces 0 items, merge behavior may drop data depending on merge mode (not explicitly shown). Validate in n8n UI.

#### Node: **Aggregate3**
- **Type/role:** Aggregate all merged items into one structure at `data[]`.
- **Output:** To **Edit Fields3**

#### Node: **Edit Fields3**
- **Type/role:** Set; concatenates three agent outputs into a single string:
  - `data = data[0].output + "\n---\n" + data[1].output + "\n---\n" + data[2].output`
- **Output:** To **LLM Critic**
- **Edge cases:**
  - Assumes exactly 3 items in `data[]` with `.output` fields.
  - If any missing, expression fails or concatenates `undefined`.

#### Node: **LLM Critic**
- **Type/role:** LangChain Agent acting as LLM-as-a-judge; produces structured critique.
- **Input text:**
  - Company name + domain
  - Research: concatenated `data`
- **System message highlights:**
  - Verify factuality “as of now”
  - Provide analysis and recommended updates to claims
- **Output parsing:** `hasOutputParser: true` via **Structured Output Parser1**
- **Model connection:** **OpenAI Chat Model3** (webSearch enabled)
- **Also connected tool:** **Think1** as an `ai_tool` (optional internal reasoning tool in n8n AI toolchain).
- **Output:** To **AI Agent**
- **Edge cases:**
  - Judge may still hallucinate corrections; web search helps but not guaranteed.
  - Parser failures if schema not followed.

#### Node: **OpenAI Chat Model3**
- **Type/role:** OpenAI provider `gpt-4o` with built-in `webSearch` enabled.

#### Node: **Structured Output Parser1**
- **Type/role:** Structured parser; schema example:
  ```json
  { "verification": ["..."], "updates": ["..."] }
  ```

#### Node: **Think1**
- **Type/role:** LangChain Think tool node wired as an AI tool into LLM Critic.
- **Role:** Provides a “thinking” tool interface; behavior depends on n8n AI internals.
- **Edge cases:** Typically none, but can be removed if not used.

---

### Block 7 — Final Dossier Agent → Format → Attio Company Update
**Overview:** Produces the final structured enrichment fields (positioning, conversation points, leadership conversations) strictly from provided inputs + judge updates, formats them into Attio fields, and upserts the Attio company record (matching by domains).

**Nodes involved:**
- AI Agent
- Think
- OpenAI Chat Model4
- Structured Output Parser2
- Code in JavaScript
- Assert Record: Company1

#### Node: **AI Agent**
- **Type/role:** LangChain Agent; final enrichment composer with structured output.
- **Input text:**
  - Discovery inputs: `{{ $('Edit Fields3').item.json.data }}`
  - Critiques: `{{ $json.output.updates }}`
- **System message highlights:**
  - Output: positioning paragraph, 1–3 conversation points, leadership conversations paragraph
  - Guardrail: MUST only use input provided
- **Output parsing:** `hasOutputParser: true` via **Structured Output Parser2**
- **Model connection:** **OpenAI Chat Model4**
- **Also connected tool:** **Think** as `ai_tool`.
- **Outputs:** To **Code in JavaScript**
- **Edge cases:** If judge output `updates` empty, still works; if research string missing, output weak.

#### Node: **Structured Output Parser2**
- **Type/role:** Structured parser; schema example:
  ```json
  {
    "positioning": "...",
    "conversation_points": "...",
    "leadership_conversations": "..."
  }
  ```

#### Node: **OpenAI Chat Model4**
- **Type/role:** OpenAI provider (`gpt-4o`).

#### Node: **Think**
- **Type/role:** Think tool node wired into AI Agent as an AI tool.

#### Node: **Code in JavaScript**
- **Type/role:** Code; maps AI Agent output into Attio company fields.
- **Key behavior:**
  - Reads `primaryDomain` from `$('SetFields').item.json.primary_domain`
  - Sets:
    - `values.positioning=[output.positioning||'']`
    - `values.conversation_points=[output.conversation_points||'']`
    - `values.leadership_conversations=[output.leadership_conversations||'']`
    - `values.enrichment_status=['complete' || '']` (note: always becomes `'complete'`)
  - Adds `values.domains=[{domain: primaryDomain}]` if domain present.
- **Outputs:** `{data:{values}}` to **Assert Record: Company1**
- **Edge cases/bugs:**
  - `['complete' || '']` is always `'complete'`; if you intended conditional completion, adjust logic.
  - Again relies on cross-node item access to `SetFields`; misalignment possible in multi-item scenarios.

#### Node: **Assert Record: Company1**
- **Type/role:** Attio upsert company record with enrichment fields.
- **Configuration:** object `companies`, match on `domains`, PUT records.
- **Failure modes:** Attio schema must include fields `positioning`, `conversation_points`, `leadership_conversations`, `enrichment_status`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Business outcome context |  |  | ## Business Outcome; Reduce time spent researching; Reinvest time in relationships |
| Sticky Note1 | Sticky Note | System outcome context |  |  | ## System Outcomes; trust outputs; minimize poor moments; human equivalence; fallbacks; graceful error handling |
| Sticky Note2 | Sticky Note | Title banner |  |  | # B2B Lead Enrichment in Attio CRM |
| Sticky Note3 | Sticky Note | Direct metric definition |  |  | ### Direct Metric; Avg Time BDR Spent Enriching Leads; baseline/target/delta |
| Sticky Note4 | Sticky Note | Proxy metrics definition |  |  | ### Proxy / Lagging Metric; opportunities won; cultivating time; deal size delta |
| Sticky Note5 | Sticky Note | Success criteria header |  |  | ## What Does Success Look Like |
| When clicking 'Execute workflow' | Manual Trigger | Start workflow manually |  | Apollo:EnrichCompany |  |
| Apollo:EnrichCompany | HTTP Request | Enrich org by domain (Apollo) | When clicking 'Execute workflow' | If1 |  |
| If1 | IF | Continue only if Apollo returned `organization` | Apollo:EnrichCompany | SetFields |  |
| SetFields | Set | Flatten Apollo organization fields | If1 | Post Assert Body: Company |  |
| Post Assert Body: Company | Code | Build Attio company upsert payload | SetFields | Assert Record: Company |  |
| Assert Record: Company | Attio | Upsert company by domain | Post Assert Body: Company | Apollo:LatestNews; Leadership Search Agent; Get a linked in company page |  |
| Apollo:LatestNews | HTTP Request | Fetch last 6 months news (Apollo) | Assert Record: Company | Split News | ## Latest News Enrichment |
| Split News | Split Out | Iterate `news_articles` | Apollo:LatestNews | Last 5 Articles | ## Latest News Enrichment |
| Last 5 Articles | Limit | Keep 5 articles | Split News | Extract | ## Latest News Enrichment |
| Extract | Tavily Extract | Extract article raw text | Last 5 Articles | Edit Fields2 | ## Latest News Enrichment |
| Edit Fields2 | Set | Keep `raw_extract` | Extract | Aggregate2 | ## Latest News Enrichment |
| Aggregate2 | Aggregate | Combine all article extracts | Edit Fields2 | LLMHelper | ## Latest News Enrichment |
| LLMHelper | Code | Format aggregated JSON as LLM prompt | Aggregate2 | Apollo Summary Agent | ## Latest News Enrichment |
| OpenAI Chat Model | OpenAI Chat Model | LLM provider for news summary agent |  | Apollo Summary Agent (as ai_languageModel) | ## Latest News Enrichment |
| Apollo Summary Agent | LangChain Agent | Summarize news into talking points | LLMHelper | Merge | ## Latest News Enrichment |
| Get a linked in company page | Scrape Creators | Scrape LinkedIn company page | Assert Record: Company | Split Out1 | ## LinkedIn Company Enrichment |
| Split Out1 | Split Out | Split company `posts` | Get a linked in company page | Last 5 Posts | ## LinkedIn Company Enrichment |
| Last 5 Posts | Limit | Keep 5 posts | Split Out1 | Extract Post Text | ## LinkedIn Company Enrichment |
| Extract Post Text | Set | Keep post `text` | Last 5 Posts | Aggregate | ## LinkedIn Company Enrichment |
| Aggregate | Aggregate | Combine post texts | Extract Post Text | LLMHelper2 | ## LinkedIn Company Enrichment |
| LLMHelper2 | Code | Format posts JSON as LLM prompt | Aggregate | LinkedIn Company Summary Agent | ## LinkedIn Company Enrichment |
| OpenAI Chat Model2 | OpenAI Chat Model | LLM provider for company summary agent |  | LinkedIn Company Summary Agent | ## LinkedIn Company Enrichment |
| LinkedIn Company Summary Agent | LangChain Agent | Summarize LinkedIn company posts | LLMHelper2 | Merge | ## LinkedIn Company Enrichment |
| GPT 4o - Date Smart | OpenAI Chat Model | LLM provider w/ web search for leadership discovery |  | Leadership Search Agent | ## Leadership Enrichment |
| Structured Output Parser | Structured Output Parser | Parse leadership list JSON |  | Leadership Search Agent | ## Leadership Enrichment |
| Leadership Search Agent | LangChain Agent | Discover leadership team (structured) | Assert Record: Company | Split People | ## Leadership Enrichment |
| Split People | Split Out | Iterate leadership_team | Leadership Search Agent | Apollo:Fetch Person | ## Leadership Enrichment |
| Apollo:Fetch Person | HTTP Request | Match leader in Apollo People | Split People | Edit Fields; Set Person Fields | ## Leadership Enrichment |
| Edit Fields | Set | Normalize person LinkedIn URL | Apollo:Fetch Person | If LinkedIn Url | ## Leadership Enrichment |
| If LinkedIn Url | IF | Only scrape if LinkedIn URL exists | Edit Fields | Get a LinkedIn Profile | ## Leadership Enrichment |
| Get a LinkedIn Profile | Scrape Creators | Scrape LinkedIn profile | If LinkedIn Url | Edit Fields1 | ## Leadership Enrichment |
| Edit Fields1 | Set | Keep profile fields for summarization | Get a LinkedIn Profile | Aggregate1 | ## Leadership Enrichment |
| Aggregate1 | Aggregate | Combine leadership profile data | Edit Fields1 | LLMHelper1 | ## Leadership Enrichment |
| LLMHelper1 | Code | Format leadership JSON as LLM prompt | Aggregate1 | LinkedIn Leadership Summary | ## Leadership Enrichment |
| OpenAI Chat Model1 | OpenAI Chat Model | LLM provider for leadership summary agent |  | LinkedIn Leadership Summary | ## Leadership Enrichment |
| LinkedIn Leadership Summary | LangChain Agent | Summarize leadership activity | LLMHelper1 | Merge | ## Leadership Enrichment |
| Set Person Fields | Set | Prepare leader fields + company domain | Apollo:Fetch Person | If Email & Domain | ## Leadership Enrichment |
| If Email & Domain | IF | Upsert person only if email domain matches company | Set Person Fields | Post Assert Body | ## Leadership Enrichment |
| Post Assert Body | Code | Build Attio person upsert payload | If Email & Domain | Assert a record | ## Leadership Enrichment |
| Assert a record | Attio | Upsert person (match by email_addresses) | Post Assert Body |  | ## Leadership Enrichment |
| Merge | Merge | Merge 3 enrichment summaries | Apollo Summary Agent; LinkedIn Company Summary Agent; LinkedIn Leadership Summary | Aggregate3 |  |
| Aggregate3 | Aggregate | Combine merged data into array | Merge | Edit Fields3 |  |
| Edit Fields3 | Set | Concatenate 3 summaries to single research text | Aggregate3 | LLM Critic |  |
| OpenAI Chat Model3 | OpenAI Chat Model | LLM provider w/ web search for judge |  | LLM Critic |  |
| Structured Output Parser1 | Structured Output Parser | Parse judge output |  | LLM Critic |  |
| Think1 | Think Tool | Tool hook for judge agent |  | LLM Critic (as ai_tool) |  |
| LLM Critic | LangChain Agent | Verify claims, propose updates | Edit Fields3 | AI Agent |  |
| OpenAI Chat Model4 | OpenAI Chat Model | LLM provider for final enrichment agent |  | AI Agent |  |
| Structured Output Parser2 | Structured Output Parser | Parse final dossier |  | AI Agent |  |
| Think | Think Tool | Tool hook for final agent |  | AI Agent (as ai_tool) |  |
| AI Agent | LangChain Agent | Produce final structured dossier | LLM Critic | Code in JavaScript |  |
| Code in JavaScript | Code | Format dossier into Attio company fields | AI Agent | Assert Record: Company1 |  |
| Assert Record: Company1 | Attio | Upsert company enrichment fields | Code in JavaScript |  |  |
| Sticky Note6 | Sticky Note | Block label |  |  | ## Leadership Enrichment |
| Sticky Note7 | Sticky Note | Block label |  |  | ## LinkedIn Company Enrichment |
| Sticky Note8 | Sticky Note | Block label |  |  | ## Latest News Enrichment |
| 35588d2c… “Edit Fields1” etc covered by labels |  |  |  |  | (labels duplicated above per node within each block) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named **“B2B Lead Enrichment - Attio CRM”**.
2. **Add Manual Trigger** node:
   - Name: *When clicking 'Execute workflow'*
3. **Add HTTP Request** node for Apollo org enrich:
   - Name: *Apollo:EnrichCompany*
   - URL: `https://api.apollo.io/api/v1/organizations/enrich`
   - Authentication: **Header Auth** (create credential “Apollo API”; header typically `X-Api-Key: <key>` depending on your Apollo setup)
   - Query param: `domain` = `{{$json.domain}}`
4. **Add IF** node:
   - Name: *If1*
   - Condition: **Object exists** on `{{$json.organization}}`
   - Connect: Manual Trigger → Apollo:EnrichCompany → If1 (true) → next step
5. **Add Set** node:
   - Name: *SetFields*
   - Map fields from `organization.*` into top-level fields (id, name, linkedin_url, primary_domain, description, etc.).
   - Important: If you keep the `replace('http','https')` expression, add a guard (recommended) to avoid null errors.
6. **Add Code** node to build Attio company payload:
   - Name: *Post Assert Body: Company*
   - Build `{ data: { values: { name: [...], description: [...], linkedin: [...], domains:[{domain:...}] } } }`
7. **Add Attio node** to upsert company:
   - Name: *Assert Record: Company*
   - Credentials: Attio API token
   - Resource: Records, Operation: PUT v2 object records
   - Object: `companies`
   - Matching attribute: `domains`
   - Data: `{{$json.data}}`
8. **Fan out from “Assert Record: Company” into 3 branches:**

### A) Latest News branch
9. Add HTTP Request:
   - Name: *Apollo:LatestNews*
   - Method: POST
   - URL: `https://api.apollo.io/api/v1/news_articles/search`
   - Auth: Apollo header auth
   - Query params:
     - `organization_ids[]` = `{{$('SetFields').item.json.id}}`
     - `published_at[min]` = `{{$now.minus(6,'months')}}`
10. Add Split Out:
    - Name: *Split News*
    - Field to split: `news_articles`
11. Add Limit:
    - Name: *Last 5 Articles*, Max items: 5
12. Add Tavily node:
    - Name: *Extract*
    - Resource: Extract
    - URLs: `{{$json.url}}`
    - Set **On Error → Continue**
    - Credential: Tavily API key
13. Add Set:
    - Name: *Edit Fields2*
    - `raw_extract = {{$json.results[0].raw_content}}`
14. Add Aggregate:
    - Name: *Aggregate2*
    - Mode: aggregate all item data
15. Add Code:
    - Name: *LLMHelper*
    - Convert `$input.all()` JSON into a single prompt string under `prompt`.
16. Add OpenAI Chat Model node:
    - Name: *OpenAI Chat Model*
    - Model: `gpt-4o`
    - Credential: OpenAI API key
17. Add LangChain Agent:
    - Name: *Apollo Summary Agent*
    - Prompt type: Define
    - Text: `{{$json.prompt}}`
    - System message: news summarizer guardrailed to org relevance
    - Connect its **ai_languageModel** input to *OpenAI Chat Model*

### B) LinkedIn company branch
18. Add Scrape Creators node:
    - Name: *Get a linked in company page*
    - Resource: LinkedIn, Operation: Get LinkedIn Company Page
    - URL: `{{$('SetFields').item.json.linkedin_url}}`
    - Credential: Scrape Creators API
19. Add Split Out:
    - Name: *Split Out1*
    - Field: `posts`
20. Add Limit:
    - Name: *Last 5 Posts*, Max: 5
21. Add Set:
    - Name: *Extract Post Text*
    - Field: `text = {{$json.text}}`
22. Add Aggregate:
    - Name: *Aggregate*
    - Aggregate all item data
23. Add Code:
    - Name: *LLMHelper2* (same pattern as other LLMHelper)
24. Add OpenAI Chat Model:
    - Name: *OpenAI Chat Model2* → `gpt-4o`
25. Add Agent:
    - Name: *LinkedIn Company Summary Agent*
    - Text: `{{$json.prompt}}`
    - System message: summarize posts into talking points
    - Connect **ai_languageModel** to *OpenAI Chat Model2*

### C) Leadership branch (discovery + profiles + optional people upsert)
26. Add OpenAI Chat Model:
    - Name: *GPT 4o - Date Smart*
    - Model: `gpt-4o`
    - Enable built-in **webSearch**
27. Add Structured Output Parser:
    - Name: *Structured Output Parser*
    - Schema example with `leadership_team[]`
28. Add Agent:
    - Name: *Leadership Search Agent*
    - Input text includes Company Name + Domain
    - Enable output parser; connect **ai_outputParser** to *Structured Output Parser*
    - Connect **ai_languageModel** to *GPT 4o - Date Smart*
29. Add Split Out:
    - Name: *Split People*
    - Field: `output.leadership_team`
30. Add HTTP Request:
    - Name: *Apollo:Fetch Person*
    - Method: POST
    - URL: `https://api.apollo.io/api/v1/people/match`
    - Auth: Apollo header auth
    - Query: `name={{$json.name}}`, `organization_name={{$('Apollo:EnrichCompany').item.json.organization.name}}`
31. **Leadership profile summarization path**
    - Add Set: *Edit Fields* to normalize `person.linkedin_url` (prefer adding a null-guard)
    - Add IF: *If LinkedIn Url* (not empty)
    - Add Scrape Creators: *Get a LinkedIn Profile* (On Error → Continue)
    - Add Set: *Edit Fields1* keep name/image/about/activity/publications
    - Add Aggregate: *Aggregate1*
    - Add Code: *LLMHelper1*
    - Add OpenAI: *OpenAI Chat Model1* (`gpt-4o`)
    - Add Agent: *LinkedIn Leadership Summary* (connect to OpenAI Chat Model1)
32. **Optional Attio People upsert path**
    - Add Set: *Set Person Fields* (name/role/email/linkedin + company domain)
    - Add IF: *If Email & Domain* (non-empty email AND email domain equals company domain)
    - Add Code: *Post Assert Body* to build Attio People payload
    - Add Attio node: *Assert a record* (match by `email_addresses`)
    - Ensure Attio object is correctly set (typically `people`) and fields exist.

### Merge + judge + final writeback
33. Add Merge:
    - Name: *Merge*
    - Inputs: 3
    - Connect the three summary agents to inputs 0/1/2.
34. Add Aggregate:
    - Name: *Aggregate3*
35. Add Set:
    - Name: *Edit Fields3*
    - Concatenate the 3 `.output` strings into one `data` string.
36. Add OpenAI Chat Model:
    - Name: *OpenAI Chat Model3* (`gpt-4o`) with **webSearch enabled**
37. Add Structured Output Parser:
    - Name: *Structured Output Parser1* with `verification[]` and `updates[]`
38. Add Think tool:
    - Name: *Think1* (optional tool for judge)
39. Add Agent:
    - Name: *LLM Critic*
    - Uses OpenAI Chat Model3
    - Uses Structured Output Parser1
    - (Optionally) connect Think1 as ai_tool
40. Add OpenAI Chat Model:
    - Name: *OpenAI Chat Model4* (`gpt-4o`)
41. Add Structured Output Parser:
    - Name: *Structured Output Parser2* with `positioning`, `conversation_points`, `leadership_conversations`
42. Add Think tool:
    - Name: *Think* (optional tool for final agent)
43. Add Agent:
    - Name: *AI Agent*
    - Input includes research + judge updates
    - Uses OpenAI Chat Model4 and Structured Output Parser2
44. Add Code:
    - Name: *Code in JavaScript*
    - Map parsed output into Attio company fields and include `domains` for matching.
45. Add Attio node:
    - Name: *Assert Record: Company1*
    - Object: `companies`
    - Matching attribute: `domains`
    - Data: `{{$json.data}}`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Business Outcome: reduce BDR research time; reinvest in relationships | Sticky note “Business Outcome” |
| System Outcomes: trust outputs; minimize incorrect moments; human-equivalent; fallbacks; graceful handling | Sticky note “System Outcomes” |
| Metrics tracking suggestions: direct time spent + lagging sales outcomes | Sticky notes “Direct Metric” and “Proxy / Lagging Metric” |
| Block labels: “Latest News Enrichment”, “LinkedIn Company Enrichment”, “Leadership Enrichment” | Sticky notes labeling node regions |
| Known implementation risks: unguarded `.replace()` on possibly-null LinkedIn URLs; item cross-references (`$('...').item`) can misalign under multi-item runs | Derived from expressions in Set/Code nodes |
| Potential bug: `enrichment_status: ['complete' || '']` always sets “complete” | In “Code in JavaScript” node |

