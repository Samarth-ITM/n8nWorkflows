Score and route new trial leads with Firecrawl, Scrapin.io, OpenAI, Slack and Instantly

https://n8nworkflows.xyz/workflows/score-and-route-new-trial-leads-with-firecrawl--scrapin-io--openai--slack-and-instantly-13644


# Score and route new trial leads with Firecrawl, Scrapin.io, OpenAI, Slack and Instantly

## 1. Workflow Overview

**Workflow name:** TOFU Sales Intelligence  
**Purpose:** Qualify and route new free-trial leads in real time by validating business email + website, extracting the company LinkedIn page, enriching company data via Scrapin.io (through an AI Agent), scoring fit (country, headcount, industry), notifying sales in Slack, and optionally adding the lead into an Instantly campaign (deduped).

### 1.1 Entry & Email Hygiene
Receives a lead via webhook, extracts the email domain, blocks personal/disposable/.edu domains, and ensures the email/domain isn’t empty.

### 1.2 Website Validation & LinkedIn Discovery
Checks the company website responds (HEAD 200–399). If healthy, scrapes the site with Firecrawl to locate the LinkedIn company URL.

### 1.3 LinkedIn Enrichment (AI Agent + Scrapin.io)
Extracts a clean LinkedIn company page URL, rejects personal profiles, enriches the company profile via Scrapin.io, and verifies completeness of returned fields.

### 1.4 Data Cleaning + Scoring Engine
Cleans the LinkedIn description for Slack, normalizes country using OpenAI, then assigns sub-scores (location/headcount/industry) and computes a final 0–100 score with a rating label.

### 1.5 Sales Notification (Slack)
Sends a rich Slack message with scores, key enrichment fields, description, and action buttons (LinkedIn + Website).

### 1.6 Score-Based Routing + Instantly Dedup + Campaign Add
Segments by score tier, searches Instantly to dedupe by email, extracts a person-like name from email (OpenAI), optionally waits, then adds lead to an Instantly campaign with a custom “website” field.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Email Handling
**Overview:** Receives inbound lead data and derives the email’s root domain. Filters out non-B2B domains and ensures required fields exist before spending credits on scraping/enrichment.

**Nodes involved:**
- Webhook
- Execution Data
- Extract Email Root Domain
- Blacklist Regex Domains
- Check to make sure email is not null

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` (Trigger)
- **Config:** POST webhook at path `/new-lead`
- **Key fields:** Expects lead payload containing `email` (note: the pinned/Execution Data suggests alternative body nesting)
- **Connections:** Outputs to **Extract Email Root Domain** and **Execution Data**
- **Edge cases:**
  - No authentication configured (note suggests adding header auth).
  - Payload shape mismatch: downstream nodes reference `$json.email`, but Execution Data references `body.data.item.email`.

#### Node: Execution Data
- **Type / role:** `n8n-nodes-base.executionData` (stores values for later debugging/replay)
- **Config:** Saves `email` from `Webhook.body.data.item.email`
- **Connections:** Triggered in parallel from Webhook; not used downstream.
- **Edge cases:** If webhook does not send that nested structure, saved email will be empty.

#### Node: Extract Email Root Domain
- **Type / role:** `n8n-nodes-base.set` (derive root domain)
- **Config:** Sets `root_domain = {{ $json.email.split('@')[1] }}`
- **Connections:** From **Webhook** → to **Blacklist Regex Domains**
- **Edge cases:**
  - If `email` missing/null → expression error or `split` failure.
  - If webhook provides `body.data.item.email` instead of `email`, this node must be updated.

#### Node: Blacklist Regex Domains
- **Type / role:** `n8n-nodes-base.if` (filter)
- **Config:** Condition: `root_domain` **notRegex** against a large list of free/disposable domains + `.*\.edu`
- **Connections:** True → **Check to make sure email is not null** (False branch not connected)
- **Edge cases / issues:**
  - There is a second condition with `leftValue: ""` equals `""` which is always true; it doesn’t block, but it’s redundant/confusing.
  - Regex maintenance: adding/removing domains requires editing the regex carefully.

#### Node: Check to make sure email is not null
- **Type / role:** `n8n-nodes-base.if` (safety gate)
- **Config:** Checks `Extract Email Root Domain.root_domain` is not empty and not the string `"null"`.
- **Connections:** True → **Check if website exists**
- **Edge cases:**
  - If Extract Email Root Domain fails earlier, this node may never run.
  - Using node-name lookups `$('Extract Email Root Domain')` can break if node renamed.

---

### Block 2 — Website Validation & Scraping
**Overview:** Validates the derived website is reachable before scraping. Scrapes the site to find a LinkedIn URL (usually in footer/header links).

**Nodes involved:**
- Check if website exists
- Check root URL
- Firecrawl Scrape

#### Node: Check if website exists
- **Type / role:** `n8n-nodes-base.httpRequest` (HEAD check)
- **Config:**
  - URL: `https://{{ $json.root_domain }}`
  - Method: `HEAD`
  - `neverError: true`, `fullResponse: true`
  - `onError: continueRegularOutput` + `retryOnFail: true`
- **Connections:** → **Check root URL**
- **Edge cases:**
  - Some sites block HEAD or redirect oddly; status codes outside 200–399 will be filtered next.
  - If `root_domain` is not a valid hostname (bad email), request may fail but “neverError” keeps output; statusCode may be missing.

#### Node: Check root URL
- **Type / role:** `n8n-nodes-base.if` (status gate)
- **Config:** Passes only if `statusCode` between 200 and 399.
- **Connections:** True → **Firecrawl Scrape**
- **Edge cases:**
  - If statusCode missing/undefined → condition fails, lead stops here.

#### Node: Firecrawl Scrape
- **Type / role:** `@mendable/n8n-nodes-firecrawl.firecrawl` (web scraping)
- **Config:**
  - Operation: `scrape`
  - URL set to `Extract Email Root Domain.root_domain` (note: no `https://` prefix in config; Firecrawl may handle it, but safer to use full URL)
  - Formats: `links` and `json` with prompt to “Return only the complete LinkedIn URL.”
  - `onlyMainContent: false` to include footer/header
- **Credentials:** Firecrawl API
- **Connections:** → **Extract LinkedIn Url**
- **Edge cases:**
  - Credit usage; large pages cost more.
  - Sites behind bot protection may yield incomplete link extraction.

---

### Block 3 — LinkedIn URL Extraction & Enrichment
**Overview:** Extracts a clean LinkedIn company page URL from Firecrawl results, rejects personal profiles, enriches via Scrapin.io through an AI Agent, and validates completeness.

**Nodes involved:**
- Extract LinkedIn Url
- Check LinkedIn is not null
- Check if Company or Personal LI Profile
- OpenAI Chat Model1
- Structured Output Parser1
- website_tool
- LinkedIn Agent
- Audit LI Results

#### Node: Extract LinkedIn Url
- **Type / role:** `n8n-nodes-base.code` (parse and normalize LinkedIn URL)
- **Config behavior:**
  - Recursively searches input JSON for first string containing `linkedin.com`
  - Normalizes to `https://www.linkedin.com/company/<slug>/` only (vanity or numeric)
  - Outputs:
    - `linkedInUrl` (normalized)
    - `hasLinkedIn` boolean
    - `debugInfo.rawLinkedInUrl`, `debugInfo.cleanLinkedInUrl`
- **Connections:** → **Check LinkedIn is not null**
- **Edge cases:**
  - If the first LinkedIn link found is not a company page (e.g., a personal profile, jobs page), it may be discarded by the regex and become null.
  - Node outputs `linkedInUrl` but downstream checks sometimes reference `debugInfo.cleanLinkedInUrl`.

#### Node: Check LinkedIn is not null
- **Type / role:** `n8n-nodes-base.if` (presence gate)
- **Config:** Verifies `{{ $json.debugInfo.cleanLinkedInUrl }}` exists, not empty, not `"null"`.
- **Connections:** True → **Check if Company or Personal LI Profile**
- **Edge cases:** If Extract LinkedIn Url returns `linkedInUrl` but not `debugInfo.cleanLinkedInUrl` as expected, the gate could fail (here it is present).

#### Node: Check if Company or Personal LI Profile
- **Type / role:** `n8n-nodes-base.if` (company-only gate)
- **Config:** Condition: `cleanLinkedInUrl` contains `linkedin.com/company`
- **Connections:** True → **LinkedIn Agent**
- **Edge cases:** Legit company pages in alternate formats (rare) could be excluded.

#### Node: OpenAI Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (LLM provider for agent)
- **Config:** Model `gpt-5.2`
- **Credentials:** OpenAI
- **Connections:** Provides `ai_languageModel` to **LinkedIn Agent**
- **Version notes:** Requires LangChain nodes installed/enabled in n8n.

#### Node: Structured Output Parser1
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (forces schema)
- **Config:** Manual JSON schema-like example for:
  - `company_name` (string)
  - `followers` (number)
  - `employee_count` (number)
  - `headquarters_location` (string)
  - `industry` (string)
  - `description` (string)
- **Connections:** Provides `ai_outputParser` to **LinkedIn Agent**
- **Edge cases:** If agent returns malformed JSON, parsing fails or fields may be missing.

#### Node: website_tool
- **Type / role:** `n8n-nodes-base.httpRequestTool` (tool callable by agent)
- **Config:**
  - GET/Query to `https://api.scrapin.io/v1/enrichment/company`
  - Query params:
    - `linkedInUrl` from agent tool input (`$fromAI(...)`)
    - `cacheDuration=4h`
  - Auth: `httpQueryAuth` (API key in query)
- **Connections:** Exposed as `ai_tool` to **LinkedIn Agent**
- **Edge cases:**
  - Scrapin.io rate limits / auth failures.
  - If the agent passes an empty LinkedIn URL, Scrapin returns error/empty results.

#### Node: LinkedIn Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` (LLM agent orchestrating tool call + structured output)
- **Config:**
  - Prompt instructs to extract only from the LinkedIn company URL and return numeric values as numbers.
  - System message: “Linkedin data analyst…use the website_tool…”
  - Input uses `{{ $json.debugInfo.cleanLinkedInUrl }}`
  - Output parser enabled (Structured Output Parser1)
  - `retryOnFail: true`
- **Connections:** → **Audit LI Results**
- **Edge cases:**
  - LinkedIn content may be incomplete or blocked; Scrapin may return partial fields.
  - If you rename `website_tool`, agent instructions must still align with the actual tool name.

#### Node: Audit LI Results
- **Type / role:** `n8n-nodes-base.if` (quality gate)
- **Config:** Checks existence of:
  - `output.company_name`
  - `output.followers` (number exists)
  - `output.employee_count` (number exists)
  - `output.headquarters_location`
- **Connections:** True → **Sanitize Description**
- **Edge cases:** Companies with sparse LinkedIn pages will be filtered out even if otherwise valuable.

---

### Block 4 — Sanitation, Normalization & Scoring
**Overview:** Makes enrichment Slack-friendly, normalizes country text into a consistent country name, scores location/headcount/industry, then computes a final weighted score and rating.

**Nodes involved:**
- Sanitize Description
- Normalize Country
- Score Country
- Score Staff Count
- Industry Scoring
- Algo Score

#### Node: Sanitize Description
- **Type / role:** `n8n-nodes-base.code` (text cleanup + field pass-through)
- **Config behavior:**
  - Pulls description from `$("LinkedIn Agent").first()?.json?.output.description` (node-name lookup)
  - Replaces smart quotes/dashes/ellipsis, removes newlines, collapses whitespace
  - Caps at 280 chars
  - Adds `sanitized_description`, and keeps existing fields
- **Connections:** → **Normalize Country**
- **Edge cases / issues:**
  - Uses node-name lookup `$("LinkedIn Agent")`: renaming that node breaks extraction.
  - Also relies on agent output being in `.json.output`; if agent output shape changes, desc becomes blank.

#### Node: Normalize Country
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI chat call)
- **Config:**
  - Model: `gpt-4.1-nano`
  - System message references `{{ $json.output.headquarters_location }}`
  - `jsonOutput: true` (expects JSON)
- **Credentials:** OpenAI
- **Connections:** → **Score Country**
- **Edge cases / issues:**
  - The prompt says “Return only the official country name or 'Unknown'”, but `jsonOutput=true` encourages JSON output. In pinned data it returned `{"country":"United States"}`. Downstream **Score Country** attempts multiple paths, but primarily expects `item.json.country` or similar; ensure you standardize this (see notes below).

#### Node: Score Country
- **Type / role:** `n8n-nodes-base.code` (0–10 location score)
- **Config behavior:**
  - Looks for country in multiple places: `item.json.country`, `item.json.normalizedCountry`, `item.json.message.content.country`, `item.json.output.country`
  - Normalizes to lowercase and matches tier lists
  - Adds `normalizedCountry` and `locationScore`
- **Connections:** → **Score Staff Count**
- **Edge cases:**
  - If Normalize Country returns a plain string (not JSON), the country may not be found unless mapped.
  - Country synonyms not listed may result in 0.

#### Node: Score Staff Count
- **Type / role:** `n8n-nodes-base.code` (0–10 headcount score)
- **Config behavior:**
  - Reads `employee_count` from the incoming item (default `sourceFieldPath = "employee_count"`)
  - Uses buckets to produce `headcountScore`
  - Outputs `staffCount` and `headcountScore`
- **Connections:** → **Industry Scoring**
- **Edge cases / issues:**
  - In this workflow, `employee_count` is actually under `output.employee_count` (from the agent). Because `sourceFieldPath` is set to `"employee_count"`, it will score **0** unless the field is flattened upstream. (Pinned execution shows this exact mismatch: employee_count=215 but headcountScore=0.)
  - Fix: set `sourceFieldPath = "output.employee_count"`.

#### Node: Industry Scoring
- **Type / role:** `n8n-nodes-base.code` (normalize industry + 0–10)
- **Config behavior:**
  - Reads industry from `sourceFieldPath = "output.industry"`
  - Maps keywords to categories + score; fallback “Other Industries” score 3
  - Outputs `normalizedIndustry`, `originalIndustry`, `industryScore`
- **Connections:** → **Algo Score**
- **Edge cases:**
  - Keyword list may miss common LinkedIn categories (e.g., “Software Development” should match Technology but current keywords might not match “software development” unless “software” catches it—here it should).

#### Node: Algo Score
- **Type / role:** `n8n-nodes-base.code` (final weighted scoring)
- **Config behavior:**
  - Reads `headcountScore`, `locationScore`, `industryScore` from incoming item
  - Applies weights (currently 1:1:1), normalizes to 0–100, assigns rating tiers
  - **Outputs only:** `{ output: { scores: ... } }` (does not preserve previous fields)
- **Connections:** Fans out to:
  - **Send to slack**
  - **Very High Value Trials (90-100)**
  - **High Value Trials (70-89)**
  - **Mid Value Trials (50-69)**
  - **Low Value Trials (0-49)**
  - **Consider adding to your CRM**
- **Edge cases / issues:**
  - Because it overwrites the item JSON, downstream nodes needing enrichment fields must use `$item(0).$node[...]` lookups (which this workflow does in Slack).
  - If any sub-score is missing, defaults to 0.

---

### Block 5 — Slack Notification
**Overview:** Posts a formatted Slack message with enrichment details, sub-scores, final score, and action buttons.

**Nodes involved:**
- Send to slack

#### Node: Send to slack
- **Type / role:** `n8n-nodes-base.slack` (message delivery)
- **Config:**
  - Channel: `brandon-automation-alerts` (ID `C08HPT21153`)
  - Message type: `block`
  - Block content includes:
    - Email from `Webhook.email`
    - Company fields from `Sanitize Description.output.*`
    - Industry category from `Industry Scoring.normalizedIndustry`
    - Subscores + final score from `Algo Score.output.scores`
    - Buttons link to LinkedIn and `https://{{ root_domain }}`
- **Credentials:** Slack OAuth/token
- **Connections:** None (notification side-effect)
- **Edge cases:**
  - Slack block JSON must stay under Slack limits (esp. long description).
  - If earlier enrichment failed, expressions referencing those nodes can error unless “Continue On Fail” is enabled (not set here).

---

### Block 6 — Segmentation + Instantly Dedup + Add to Campaign
**Overview:** Routes leads based on score tier, searches Instantly to avoid duplicates, extracts a name from email, optionally waits, and adds to a campaign.

**Nodes involved:**
- Very High Value Trials (90-100)
- High Value Trials (70-89)
- Mid Value Trials (50-69)
- Low Value Trials (0-49)
- Search Instantly Database3 / 2 / 1 / (base)
- Check if in db / 1 / 2 / 3
- Extract Name From Email / 2 / 3 / 4
- Wait / Wait2 / Wait3 / Wait4
- Add lead to campaign1 / 3 / 2 / (base)
- Consider adding to your CRM

#### Node: Very High Value Trials (90-100)
- **Type / role:** `n8n-nodes-base.if`
- **Config:** `finalScore >= 90`
- **Connections:** True → **Search Instantly Database3**

#### Node: High Value Trials (70-89)
- **Type / role:** `n8n-nodes-base.if`
- **Config:** `70 <= finalScore <= 89`
- **Connections:** True → **Search Instantly Database2**

#### Node: Mid Value Trials (50-69)
- **Type / role:** `n8n-nodes-base.if`
- **Config:** `50 <= finalScore <= 69`
- **Connections:** True → **Search Instantly Database1**

#### Node: Low Value Trials (0-49)
- **Type / role:** `n8n-nodes-base.if`
- **Config:** `0 <= finalScore <= 49`
- **Connections:** True → **Search Instantly Database**

#### Nodes: Search Instantly Database / 1 / 2 / 3
- **Type / role:** `CUSTOM.instantly` (custom/community Instantly node)
- **Config:** Resource `lead`, operation `getMany`, filter search = webhook email
- **Credentials:** Various Instantly accounts (note: node 3 uses different credential name than others)
- **Connections:** Each flows to its corresponding **Check if in db*** node
- **Edge cases:**
  - API limits/auth failures stop that branch.
  - Response shape expected: `$json.items` array.

#### Nodes: Check if in db / 1 / 2 / 3
- **Type / role:** `n8n-nodes-base.if` (dedup gate)
- **Config:** Condition: `$json.items` is **empty** (meaning lead not found)  
  (This is slightly counterintuitive: “empty” => OK to add.)
- **Connections:** True → corresponding **Extract Name From Email*** node
- **Edge cases:** If Instantly returns null or different structure, the “empty” check may behave unexpectedly.

#### Nodes: Extract Name From Email / 2 / 3 / 4
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (name parsing)
- **Config:**
  - Model: `gpt-4.1-nano`
  - Prompts: extract Name/First Name/Last Name from local part; if uncertain output “there”
  - Uses `{{ $item("0").$node["Webhook"].json["email"] }}`
  - `jsonOutput: true`
- **Connections:** → corresponding **Wait*** node
- **Edge cases:**
  - If webhook email is nested (body), expression may fail.
  - Output paths used later assume structure `message.content['First Name']`; confirm actual output shape in your n8n version.

#### Nodes: Wait / Wait2 / Wait3 / Wait4
- **Type / role:** `n8n-nodes-base.wait` (optional delay)
- **Config:**
  - `unit: minutes`
  - Wait4 is **disabled** and set to 20 minutes; others have no amount specified (defaults depend on n8n UI—should be set explicitly)
- **Connections:** → corresponding **Add lead to campaign*** node
- **Edge cases:** Wait nodes require workflow to be active and n8n to have execution persistence configured.

#### Nodes: Add lead to campaign1 / 3 / 2 / Add lead to campaign
- **Type / role:** `CUSTOM.instantly` (add to campaign)
- **Config:**
  - Operation: `addToCampaign`
  - Email from webhook
  - Campaign: `TEST` (same campaign ID in all four)
  - First/Last name pulled from corresponding Extract Name From Email node:  
    `$('Extract Name From EmailX').item.json.message.content['First Name']`
  - Custom field: `website = https://{{ Check to make sure email is not null.root_domain }}`
- **Credentials:** “Instantly Partnerships”
- **Connections:** None
- **Edge cases / issues:**
  - If name extraction returns “there” as a single token, last name may be empty.
  - Custom field path uses the IF node output; ensure `root_domain` is actually present on that branch’s item (it should be, but verify with execution data).
  - You likely want different campaign IDs per tier (currently all point to the same).

#### Node: Consider adding to your CRM
- **Type / role:** `n8n-nodes-base.noOp` (placeholder)
- **Config:** None
- **Connections:** From Algo Score (parallel)
- **Use:** Intended insertion point to push to HubSpot/Salesforce/etc.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Entry trigger for new leads | — | Extract Email Root Domain; Execution Data | # n8n Lead Qualification & Sales Intelligence Workflow (Top of Funnel)… (full overview + links: https://www.youtube.com/@brandoncharleson, https://www.linkedin.com/in/brandon-charleson, https://x.com/brandon_ai) |
| Execution Data | n8n-nodes-base.executionData | Persist email for debugging/replay | Webhook | — | # n8n Lead Qualification & Sales Intelligence Workflow (Top of Funnel)… (full overview + links) |
| Extract Email Root Domain | n8n-nodes-base.set | Derive domain from email | Webhook | Blacklist Regex Domains | ## Email handling — This extracts the root domain and omits personal/trash emails |
| Blacklist Regex Domains | n8n-nodes-base.if | Filter out personal/disposable/.edu domains | Extract Email Root Domain | Check to make sure email is not null | ## Email handling — This extracts the root domain and omits personal/trash emails |
| Check to make sure email is not null | n8n-nodes-base.if | Safety gate for missing/empty root_domain | Blacklist Regex Domains | Check if website exists | ## Email handling — This extracts the root domain and omits personal/trash emails |
| Check if website exists | n8n-nodes-base.httpRequest | HEAD request to validate domain | Check to make sure email is not null | Check root URL | ## Check if website exists and scrape — This does a HEAD request to check if successful. If successful then scrapes |
| Check root URL | n8n-nodes-base.if | Continue only if HTTP 200–399 | Check if website exists | Firecrawl Scrape | ## Check if website exists and scrape — This does a HEAD request to check if successful. If successful then scrapes |
| Firecrawl Scrape | @mendable/n8n-nodes-firecrawl.firecrawl | Scrape site to find LinkedIn URL | Check root URL | Extract LinkedIn Url | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| Extract LinkedIn Url | n8n-nodes-base.code | Parse and normalize LinkedIn company URL | Firecrawl Scrape | Check LinkedIn is not null | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| Check LinkedIn is not null | n8n-nodes-base.if | Guard: LinkedIn URL exists | Extract LinkedIn Url | Check if Company or Personal LI Profile | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| Check if Company or Personal LI Profile | n8n-nodes-base.if | Allow only linkedin.com/company pages | Check LinkedIn is not null | LinkedIn Agent | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for Agent | — | LinkedIn Agent (ai_languageModel) | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured agent output | — | LinkedIn Agent (ai_outputParser) | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| website_tool | n8n-nodes-base.httpRequestTool | Agent tool calling Scrapin.io enrichment | — | LinkedIn Agent (ai_tool) | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| LinkedIn Agent | @n8n/n8n-nodes-langchain.agent | Enrich LinkedIn company data via tool | Check if Company or Personal LI Profile | Audit LI Results | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| Audit LI Results | n8n-nodes-base.if | Ensure required enrichment fields exist | LinkedIn Agent | Sanitize Description | ## Extract company LinkedIn URL & enrich company info — …Uses scrapin.io as an enrichment tool for the AI Agent. |
| Sanitize Description | n8n-nodes-base.code | Clean description for Slack + pass fields | Audit LI Results | Normalize Country | ## Sanitize data for Slack, run scoring algo checks for qual and quant scores (based on ICP). |
| Normalize Country | @n8n/n8n-nodes-langchain.openAi | Convert location string → country | Sanitize Description | Score Country | ## Sanitize data for Slack, run scoring algo checks for qual and quant scores (based on ICP). |
| Score Country | n8n-nodes-base.code | Assign 0–10 location score | Normalize Country | Score Staff Count | ## Sanitize data for Slack, run scoring algo checks for qual and quant scores (based on ICP). |
| Score Staff Count | n8n-nodes-base.code | Assign 0–10 headcount score | Score Country | Industry Scoring | ## Sanitize data for Slack, run scoring algo checks for qual and quant scores (based on ICP). |
| Industry Scoring | n8n-nodes-base.code | Normalize industry + assign 0–10 score | Score Staff Count | Algo Score | ## Sanitize data for Slack, run scoring algo checks for qual and quant scores (based on ICP). |
| Algo Score | n8n-nodes-base.code | Combine subscores → final 0–100 + rating | Industry Scoring | Send to slack; Very High Value Trials; High Value Trials; Mid Value Trials; Low Value Trials; Consider adding to your CRM | ## Sanitize data for Slack, run scoring algo checks for qual and quant scores (based on ICP). |
| Send to slack | n8n-nodes-base.slack | Sales notification with rich blocks | Algo Score | — | ## Sanitize data for Slack, run scoring algo checks for qual and quant scores (based on ICP). |
| Very High Value Trials (90-100) | n8n-nodes-base.if | Route score tier 90+ | Algo Score | Search Instantly Database3 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| High Value Trials (70-89) | n8n-nodes-base.if | Route score tier 70–89 | Algo Score | Search Instantly Database2 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Mid Value Trials (50-69) | n8n-nodes-base.if | Route score tier 50–69 | Algo Score | Search Instantly Database1 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Low Value Trials (0-49) | n8n-nodes-base.if | Route score tier 0–49 | Algo Score | Search Instantly Database | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Search Instantly Database3 | CUSTOM.instantly | Dedupe lookup (very high tier) | Very High Value Trials (90-100) | Check if in db | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Check if in db | n8n-nodes-base.if | Proceed only if not found (items empty) | Search Instantly Database3 | Extract Name From Email | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Extract Name From Email | @n8n/n8n-nodes-langchain.openAi | Parse first/last name from email | Check if in db | Wait | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Wait | n8n-nodes-base.wait | Optional delay | Extract Name From Email | Add lead to campaign1 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Add lead to campaign1 | CUSTOM.instantly | Add lead to Instantly campaign | Wait | — | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Search Instantly Database2 | CUSTOM.instantly | Dedupe lookup (high tier) | High Value Trials (70-89) | Check if in db1 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Check if in db1 | n8n-nodes-base.if | Proceed only if not found | Search Instantly Database2 | Extract Name From Email2 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Extract Name From Email2 | @n8n/n8n-nodes-langchain.openAi | Parse name from email | Check if in db1 | Wait2 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Wait2 | n8n-nodes-base.wait | Optional delay | Extract Name From Email2 | Add lead to campaign3 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Add lead to campaign3 | CUSTOM.instantly | Add lead to Instantly campaign | Wait2 | — | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Search Instantly Database1 | CUSTOM.instantly | Dedupe lookup (mid tier) | Mid Value Trials (50-69) | Check if in db2 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Check if in db2 | n8n-nodes-base.if | Proceed only if not found | Search Instantly Database1 | Extract Name From Email3 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Extract Name From Email3 | @n8n/n8n-nodes-langchain.openAi | Parse name from email | Check if in db2 | Wait3 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Wait3 | n8n-nodes-base.wait | Optional delay | Extract Name From Email3 | Add lead to campaign2 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Add lead to campaign2 | CUSTOM.instantly | Add lead to Instantly campaign | Wait3 | — | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Search Instantly Database | CUSTOM.instantly | Dedupe lookup (low tier) | Low Value Trials (0-49) | Check if in db3 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Check if in db3 | n8n-nodes-base.if | Proceed only if not found | Search Instantly Database | Extract Name From Email4 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Extract Name From Email4 | @n8n/n8n-nodes-langchain.openAi | Parse name from email | Check if in db3 | Wait4 | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Wait4 | n8n-nodes-base.wait | Optional delay (disabled) | Extract Name From Email4 | Add lead to campaign | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Add lead to campaign | CUSTOM.instantly | Add lead to Instantly campaign | Wait4 | — | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Consider adding to your CRM | n8n-nodes-base.noOp | Placeholder for CRM integration | Algo Score | — | ## Segment scores, verify lead is not already in Instantly, then add to campaign |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named **“TOFU Sales Intelligence”**.

2. **Add Webhook (Trigger)**
   - Node: **Webhook**
   - Method: **POST**
   - Path: **new-lead**
   - (Recommended) Add header auth or some shared secret validation (not present by default in this workflow).
   - Expected input should include an `email` field at top level, or you must adjust downstream expressions.

3. **(Optional) Add Execution Data**
   - Node: **Execution Data**
   - Save key `email` from your webhook’s actual path (e.g., `{{$json.email}}` or `{{$json.body.data.item.email}}`).

4. **Add “Extract Email Root Domain” (Set)**
   - Create a Set node and add field:
     - `root_domain` (string) = `{{ $json.email.split('@')[1] }}`

5. **Add “Blacklist Regex Domains” (IF)**
   - Condition: String → **does not match regex**
   - Left: `{{ $json.root_domain }}`
   - Right: your blacklist regex (include free/disposable providers + `.edu`)
   - Connect True output to the next step.

6. **Add “Check to make sure email is not null” (IF)**
   - Conditions (AND):
     - `{{ $('Extract Email Root Domain').item.json.root_domain }}` **is not empty**
     - and **not equals** `"null"`
   - True → next step

7. **Add “Check if website exists” (HTTP Request)**
   - Method: **HEAD**
   - URL: `https://{{ $json.root_domain }}`
   - Response options: enable **Full Response**, enable **Never Error**
   - Set **On Error** to continue (or handle explicitly)
   - True output goes to next IF.

8. **Add “Check root URL” (IF)**
   - Conditions:
     - `statusCode >= 200`
     - `statusCode <= 399`
   - True → Firecrawl node

9. **Add “Firecrawl Scrape”**
   - Operation: **Scrape**
   - URL: ideally `https://{{ $json.root_domain }}` (safer than bare domain)
   - Output formats:
     - `links`
     - `json` with prompt: “Find the LinkedIn company profile URL…Return only the complete LinkedIn URL.”
   - Add Firecrawl credentials.

10. **Add “Extract LinkedIn Url” (Code)**
   - Paste the code logic to recursively locate the first LinkedIn URL and normalize to `/company/<slug>/`.
   - Ensure it outputs `debugInfo.cleanLinkedInUrl` and/or `linkedInUrl`.

11. **Add “Check LinkedIn is not null” (IF)**
   - Check that `{{ $json.debugInfo.cleanLinkedInUrl }}` exists and is not empty/null.

12. **Add “Check if Company or Personal LI Profile” (IF)**
   - Condition: contains `linkedin.com/company`
   - True → AI enrichment

13. **Add Scrapin tool node: “website_tool” (HTTP Request Tool)**
   - URL: `https://api.scrapin.io/v1/enrichment/company`
   - Query params:
     - `linkedInUrl` (value supplied by the agent)
     - `cacheDuration=4h`
   - Auth: HTTP Query Auth (Scrapin.io API key)
   - This must be an **HTTP Request Tool** node (not a normal HTTP Request) so the agent can call it.

14. **Add LangChain nodes for the Agent**
   - **OpenAI Chat Model** node (LM): pick `gpt-5.2` (or your choice), set OpenAI credentials.
   - **Structured Output Parser** node: define the output schema fields (company_name, followers, employee_count, headquarters_location, industry, description).
   - **Agent** node:
     - System message instructing extraction only
     - Prompt includes the LinkedIn URL from `{{ $json.debugInfo.cleanLinkedInUrl }}`
     - Attach the **OpenAI Chat Model** as language model input
     - Attach **website_tool** as tool input
     - Attach the **Structured Output Parser** as output parser
     - Enable retry on fail (optional)

15. **Add “Audit LI Results” (IF)**
   - Ensure the agent output has key fields (company_name, followers, employee_count, headquarters_location).

16. **Add “Sanitize Description” (Code)**
   - Clean description for Slack and add `sanitized_description`.
   - Prefer using **incoming item data** (`item.json.output.description`) rather than `$("LinkedIn Agent")` lookups to avoid breakage when renaming.

17. **Add “Normalize Country” (OpenAI node)**
   - Model: `gpt-4.1-nano`
   - Provide `headquarters_location` and request a standardized country.
   - Decide one consistent output format:
     - Either return plain text country, or JSON `{ "country": "United States" }`.
   - Keep `jsonOutput` aligned with your prompt.

18. **Add “Score Country” (Code)**
   - Implement tiered country lists and output `locationScore` 0–10.

19. **Add “Score Staff Count” (Code)**
   - IMPORTANT: set `sourceFieldPath` to match your data shape:
     - Use `output.employee_count` if agent output is nested.
   - Output `headcountScore`.

20. **Add “Industry Scoring” (Code)**
   - `sourceFieldPath = output.industry`
   - Output `industryScore` and `normalizedIndustry`.

21. **Add “Algo Score” (Code)**
   - Combine the three sub-scores into `finalScore` 0–100 and a `rating`.
   - If you want to keep upstream fields, spread `...item.json` into output rather than overwriting everything.

22. **Add “Send to slack” (Slack)**
   - Operation: post message to a channel
   - Use block message format; reference values from earlier nodes using `$item(0).$node[...]`.
   - Add Slack credentials and pick channel.

23. **Add Score Tier IF nodes**
   - Very High: `finalScore >= 90`
   - High: `70–89`
   - Mid: `50–69`
   - Low: `0–49`

24. **For each tier, add Instantly dedupe + add**
   - **Search Instantly Database (CUSTOM.instantly)**: `lead.getMany` with search = email
   - **Check if in db (IF)**: `$json.items` is empty
   - **Extract Name From Email (OpenAI)**: output Name/First/Last or “there”
   - **Wait** (optional): set minutes explicitly
   - **Add lead to campaign (CUSTOM.instantly)**:
     - Operation: `addToCampaign`
     - Campaign ID per tier (recommended)
     - Custom field: `website = https://{{ root_domain }}`

25. **Add placeholder “Consider adding to your CRM” (NoOp)**
   - Connect from Algo Score for future HubSpot/Salesforce insertion.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Recommend adding a header auth to protect your endpoint.” | Webhook node note |
| Built by Brandon Charleson | Included in main sticky note |
| Contact: brandon@topoffunnel.com | Included in main sticky note |
| YouTube: https://www.youtube.com/@brandoncharleson | Included in main sticky note |
| LinkedIn: https://www.linkedin.com/in/brandon-charleson | Included in main sticky note |
| X: https://x.com/brandon_ai | Included in main sticky note |
| Key implementation caution: Score Staff Count reads `employee_count` but agent output is `output.employee_count` | Fix by changing `sourceFieldPath` to `output.employee_count` to avoid always scoring 0 |
| Key implementation caution: webhook email path must be consistent (`$json.email` vs nested body) | Align expressions early (Webhook → Set node) to avoid null splits/errors |