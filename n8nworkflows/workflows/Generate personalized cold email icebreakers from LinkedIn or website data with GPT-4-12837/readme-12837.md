Generate personalized cold email icebreakers from LinkedIn or website data with GPT-4

https://n8nworkflows.xyz/workflows/generate-personalized-cold-email-icebreakers-from-linkedin-or-website-data-with-gpt-4-12837


# Generate personalized cold email icebreakers from LinkedIn or website data with GPT-4

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate personalized cold email icebreakers from LinkedIn or website data with GPT-4

**Purpose:**  
Automate personalized cold email “opening blocks” (icebreaker + intro + inferred company type) for leads stored in Google Sheets. For each lead, the workflow first tries to scrape recent LinkedIn posts via Apify; if no posts are found, it falls back to scraping the company website. In both cases it summarizes context with GPT-4 and then generates concise email copy with GPT-4, finally updating the original Google Sheet row.

**Primary use cases**
- Sales/BD personalization at scale for outbound email.
- Enrichment pipeline producing short “icebreaker” and “intro” fields.
- Automated fallback logic when LinkedIn content is unavailable.

### 1.1 Input Reception & Lead Selection
Manual trigger → read leads from Google Sheets (filtered by missing/target column).

### 1.2 Per-Lead Loop & Field Mapping
Split items into batches → normalize lead fields into a consistent schema (names, URLs, website, etc.).

### 1.3 LinkedIn Scrape Attempt (Apify)
Call Apify actor to fetch recent LinkedIn posts for the lead’s LinkedIn URL(s).

### 1.4 Conditional Branching
If Apify indicates “No posts found.” → website fallback branch; else → LinkedIn branch.

### 1.5 Website Fallback Branch
HTTP fetch company website → convert HTML to Markdown → GPT summarizes website/person context → GPT writes email copy → update Google Sheet row.

### 1.6 LinkedIn Branch
Aggregate scraped posts → GPT summarizes LinkedIn/person context → GPT writes email copy → update Google Sheet row.

### 1.7 Loop Continuation
Merge both branches into a single path and feed back into the batch loop to process the next lead.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Lead Fetch
**Overview:** Starts the workflow manually and loads lead rows from Google Sheets, filtering to rows requiring processing.  
**Nodes Involved:**  
- When clicking ‘Execute workflow’
- Fetch Leads

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — interactive start for testing or ad-hoc runs.
- **Configuration choices:** No parameters.
- **Connections:**
  - **Output →** Fetch Leads
- **Edge cases / failures:** None (manual trigger).
- **Version notes:** `typeVersion: 1` (standard).

#### Node: Fetch Leads
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads rows from a sheet.
- **Configuration (interpreted):**
  - Targets a spreadsheet (`YOUR_GOOGLE_SHEET_ID`) and a sheet/tab (`YOUR_SHEET_GID`, shown as “Sheet1” in cached metadata).
  - Uses **Filters UI** with a lookup column **Icebreaker**. (In practice, this is typically used to filter blank or specific values; the JSON shows the lookup column but not the comparator/value, so confirm in node UI.)
- **Key fields expected in sheet (per notes / mappings):**
  - `email_final`, `linkedin_url`, `organization_website_url` (or equivalent), plus other enrichment fields used later.
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account”).
- **Connections:**
  - **Output →** Loop Over Leads
- **Edge cases / failures:**
  - Auth/consent failure (OAuth).
  - Spreadsheet permissions, wrong sheet/tab IDs.
  - Filter misconfiguration leading to zero rows or unintended rows.
- **Version notes:** `typeVersion: 4.6` (be mindful of UI differences vs older Google Sheets node versions).

---

### Block 2 — Per-Lead Iteration & Normalization
**Overview:** Processes leads one at a time (or per batch) and maps varying sheet column names into a stable JSON schema used downstream.  
**Nodes Involved:**  
- Loop Over Leads
- Map Lead Data

#### Node: Loop Over Leads
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates through items to avoid overloading downstream services.
- **Configuration choices:**
  - Batch options are empty in JSON; default batch size in UI applies unless set.
  - The workflow uses the node’s second output (“continue”) to drive per-item processing.
- **Connections:**
  - **Input ←** Fetch Leads
  - **Output (main index 1) →** Map Lead Data
  - **Input also receives loopback ←** Merge Loops (to continue with next batch item)
- **Edge cases / failures:**
  - If batch size is too large, may hit Apify/OpenAI/Google rate limits.
  - If loopback path is broken, it will process only the first batch.
- **Version notes:** `typeVersion: 3`

#### Node: Map Lead Data
- **Type / role:** Set (`n8n-nodes-base.set`) — field mapping/normalization.
- **Configuration (interpreted):** Creates normalized fields from sheet columns:
  - `firstName, lastName` = `first_name`, `last_name` concatenated with comma
  - `email` = `email_final`
  - `city`, `state`, `country` passthrough
  - `companyWebsite` = `organization_website_url`
  - `companyName` = `organization_name`
  - `keywords` = `keywords`
  - `contactLinkedIn` = `linkedin_url`
  - `companyLinkedIn` = `organization_linkedin_url`
  - `organizationDescription` = `organization_short_description`
- **Key expressions / variables:**
  - Uses expressions like `={{ $json.first_name }}` (direct mapping from current item).
- **Connections:**
  - **Input ←** Loop Over Leads
  - **Output →** Apify LinkedIn Scraper
- **Edge cases / failures:**
  - Missing columns in the sheet → empty strings propagate and can cause Apify targetUrls to be empty or website scrape to fail.
  - The field name `firstName, lastName` includes a comma; this is valid in n8n but can be awkward in downstream references (workflow already references it with bracket notation).
- **Version notes:** `typeVersion: 3.4`

---

### Block 3 — LinkedIn Scrape Attempt (Apify)
**Overview:** Calls an Apify actor synchronously to fetch up to 5 recent posts from one or more LinkedIn URLs.  
**Nodes Involved:**  
- Apify LinkedIn Scraper

#### Node: Apify LinkedIn Scraper
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — invokes Apify API.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.apify.com/v2/acts/A3cAPGpwBEG8RJwse/run-sync-get-dataset-items`
    - This endpoint runs the actor synchronously and returns dataset items.
  - **Authorization header:** `Bearer YOUR_TOKEN_HERE` (placeholder)
  - **JSON body:** Configures scrape behavior:
    - `includeQuotePosts: true`, `includeReposts: false`
    - Max limits: `maxPosts: 5`, `maxComments: 5`, `maxReactions: 5`
    - `scrapeComments: false`, `scrapeReactions: false`
    - `targetUrls`: built from `contactLinkedIn`
      - If `contactLinkedIn` is an array → use it
      - Else split by comma, trim, filter empties
      - Wrapped with `JSON.stringify(...)` to be valid JSON array
  - **Reliability settings:**
    - `retryOnFail: true`, `maxTries: 5`, `waitBetweenTries: 5000ms`
- **Connections:**
  - **Input ←** Map Lead Data
  - **Output →** Posts Found?
- **Edge cases / failures:**
  - Invalid/expired Apify token (401/403).
  - Actor ID or endpoint changes.
  - LinkedIn anti-bot blocks leading to empty dataset or actor errors.
  - If `contactLinkedIn` is blank/invalid, actor may return an error or no items.
  - Response shape must include a `message` field for the next IF check; if Apify returns items without `message`, the IF condition may misroute (see next block).
- **Version notes:** `typeVersion: 4.2`

---

### Block 4 — Branch Decision: Posts vs No Posts
**Overview:** Checks whether the Apify response indicates no posts and routes either to website fallback or LinkedIn processing.  
**Nodes Involved:**  
- Posts Found?

#### Node: Posts Found?
- **Type / role:** IF (`n8n-nodes-base.if`) — conditional routing.
- **Configuration (interpreted):**
  - Condition: `{{ $json.message }}` **equals** `"No posts found."`
  - If condition is **true** (no posts) → website branch
  - If condition is **false** (posts exist) → LinkedIn branch
- **Connections:**
  - **Input ←** Apify LinkedIn Scraper
  - **True output (index 0) →** Scrape Company Website
  - **False output (index 1) →** Aggregate Posts
- **Edge cases / failures:**
  - If Apify returns an array of posts without `message`, `$json.message` may be `undefined`; condition becomes false → the workflow will assume posts exist and proceed to Aggregate Posts (may still work if items are posts).
  - If Apify returns a different “no posts” indicator, the condition will not match.
- **Version notes:** `typeVersion: 2.2` with conditions “version 2” semantics (stricter typing options shown).

---

### Block 5 — Website Fallback Branch (Scrape → Summarize → Write Copy → Update Sheet)
**Overview:** Scrapes the company website, converts HTML to Markdown, summarizes key context using GPT, writes email copy using GPT, then updates the lead row in Google Sheets.  
**Nodes Involved:**  
- Scrape Company Website
- Convert HTML to Markdown
- Analyze Website Context
- Write Email Copy (Website Context)
- Update Row (Website Data)
- Pass-through (Website)

#### Node: Scrape Company Website
- **Type / role:** HTTP Request — fetches the raw website HTML.
- **Configuration (interpreted):**
  - **URL:** `{{ $('Map Lead Data').item.json.companyWebsite }}`
  - **Error handling:** `onError: continueErrorOutput`
    - If the request fails (DNS, 4xx/5xx, timeout), it continues via the error output (but note: downstream nodes are connected to the regular output; depending on n8n behavior, you may still get an item with error fields or may need explicit error-output wiring).
- **Connections:**
  - **Input ←** Posts Found? (true path)
  - **Output →** Convert HTML to Markdown
- **Edge cases / failures:**
  - Empty/invalid URL, redirects to blocked pages, bot protection, large pages.
  - Non-HTML response types (PDF, JS app shells) reduce usefulness.
- **Version notes:** `typeVersion: 4.2`

#### Node: Convert HTML to Markdown
- **Type / role:** Markdown (`n8n-nodes-base.markdown`) — converts HTML content to Markdown/plain text for easier LLM ingestion.
- **Configuration (interpreted):**
  - **Input HTML:** `{{ $json.data }}`
  - **Error handling:** `onError: continueRegularOutput`
    - If conversion fails, it continues with original item; LLM may receive empty/invalid `data`.
- **Connections:**
  - **Input ←** Scrape Company Website
  - **Output →** Analyze Website Context
- **Edge cases / failures:**
  - If `data` is missing (from a failed HTTP request), conversion yields empty output or errors.
- **Version notes:** `typeVersion: 1`

#### Node: Analyze Website Context
- **Type / role:** OpenAI (LangChain) (`@n8n/n8n-nodes-langchain.openAi`) — summarizes website + person data into a strict JSON schema.
- **Configuration (interpreted):**
  - **Model:** `gpt-4-turbo`
  - **Temperature:** 0.6
  - **JSON output enabled:** `jsonOutput: true` (expects a parseable JSON object)
  - Prompts:
    - System: “DO NOT MAKE UP FALSE DATA”
    - User: instructs to output:
      - `WebsiteContext`, `PersonContext`, `UniqueAngles`
    - Includes website scrape truncated to 5000 chars:
      - `{{ $json.data.length > 5000 ? $json.data.slice(0, 5000) : $json.data }}`
    - Includes lead fields from **Map Lead Data** via `$('Map Lead Data').item.json...`
- **Connections:**
  - **Input ←** Convert HTML to Markdown
  - **Output →** Write Email Copy (Website Context)
- **Edge cases / failures:**
  - Model access issues (permissions/billing), timeouts, token limits.
  - If website scrape is extremely boilerplate, output may be “no content”.
  - If the LLM outputs non-JSON (despite jsonOutput), node may error.
- **Version notes:** `typeVersion: 1.8` (LangChain OpenAI node behavior can differ across versions).

#### Node: Write Email Copy (Website Context)
- **Type / role:** OpenAI (LangChain) — generates the email building blocks JSON.
- **Configuration (interpreted):**
  - **Model:** `gpt-4`
  - **Temperature:** 0.7
  - **JSON output enabled**
  - Prompts enforce:
    - No em dashes
    - Spartan, laconic, familiar tone; first-person “I”
    - Output JSON: `icebreaker`, `intro`, `companyType`
  - Context injected from prior node:
    - `Website Context: {{ $json.message.content.WebsiteContext }}`
    - `Person Context: {{ $json.message.content.PersonContext }}`
    - `Unique Angles: {{ $json.message.content.UniqueAngles }}`
  - Includes a couple of example assistant messages (few-shot style) embedded in the prompt history.
- **Connections:**
  - **Input ←** Analyze Website Context
  - **Output →** Update Row (Website Data)
- **Edge cases / failures:**
  - If Analyze Website Context output structure differs, expressions like `$json.message.content.WebsiteContext` can break.
  - LLM may still produce em dashes unless strongly constrained; consider post-processing validation.
- **Version notes:** `typeVersion: 1.8`

#### Node: Update Row (Website Data)
- **Type / role:** Google Sheets — updates the row for the processed lead.
- **Configuration (interpreted):**
  - **Operation:** Update
  - **Match row by:** `email_final`
  - **Values written:**
    - `intro` = `{{ $json.message.content.intro }}`
    - `Icebreaker` = `{{ $json.message.content.icebreaker }}`
    - `companyType` = `{{ $json.message.content.companyType }}`
    - `email_final` = `{{ $('Fetch Leads').item.json.email_final }}`
  - **Retry:** enabled, wait 5s between tries.
- **Connections:**
  - **Input ←** Write Email Copy (Website Context)
  - **Output →** Pass-through (Website)
- **Edge cases / failures:**
  - If `email_final` is not unique, the wrong row may be updated.
  - If `email_final` is blank, update will fail or update unintended rows depending on Sheets behavior.
  - Rate limits on Sheets API.
- **Version notes:** `typeVersion: 4.6`

#### Node: Pass-through (Website)
- **Type / role:** Set — no assignments; used to standardize flow and ensure output continues even if prior node behavior changes.
- **Configuration:** Empty Set node; `alwaysOutputData: true`.
- **Connections:**
  - **Input ←** Update Row (Website Data)
  - **Output →** Merge Loops (input 0)
- **Edge cases / failures:** None meaningful.
- **Version notes:** `typeVersion: 3.4`

---

### Block 6 — LinkedIn Branch (Aggregate → Summarize → Write Copy → Update Sheet)
**Overview:** Aggregates up to 5 scraped LinkedIn posts into a compact structure, summarizes them with GPT, writes personalized email copy, then updates the sheet.  
**Nodes Involved:**  
- Aggregate Posts
- Analyze LinkedIn Context
- Write Email Copy (LinkedIn Context)
- Update Row (LinkedIn Data)
- Pass-through (LinkedIn)

#### Node: Aggregate Posts
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — combines multiple post items into aggregated fields.
- **Configuration (interpreted):**
  - Aggregates fields:
    - `type`
    - `content` (renamed from `content`)
    - `repostContent` (renamed from `repost.content`)
  - Output becomes a single item containing arrays (or aggregated values depending on node settings defaults).
- **Connections:**
  - **Input ←** Posts Found? (false path)
  - **Output →** Analyze LinkedIn Context
- **Edge cases / failures:**
  - If Apify output doesn’t contain these fields, aggregates may be empty.
  - Some Apify datasets can nest content differently; verify actual field names.
- **Version notes:** `typeVersion: 1`

#### Node: Analyze LinkedIn Context
- **Type / role:** OpenAI (LangChain) — condenses LinkedIn posts into JSON (`LinkedInContext`, `PersonContext`, `UniqueAngles`).
- **Configuration (interpreted):**
  - **Model:** `gpt-4`
  - **Temperature:** 0.6
  - **JSON output enabled**
  - Prompts:
    - Strict instruction not to invent data.
    - Input includes:
      - `{{ $json.content }}` and `{{ $json.repostContent }}`
      - Lead metadata from Map Lead Data (name/location/company/keywords/description).
- **Connections:**
  - **Input ←** Aggregate Posts
  - **Output →** Write Email Copy (LinkedIn Context)
- **Edge cases / failures:**
  - If aggregated `content` is an array, the prompt receives an array string representation; may be okay but consider joining for readability.
  - Same LLM JSON validity concerns as above.
- **Version notes:** `typeVersion: 1.8`

#### Node: Write Email Copy (LinkedIn Context)
- **Type / role:** OpenAI (LangChain) — generates the email opening blocks based on LinkedIn-derived context.
- **Configuration (interpreted):**
  - **Model:** `gpt-4`
  - **Temperature:** 0.7
  - **JSON output enabled**
  - Prompts enforce:
    - No em dashes
    - Spartan tone, first-person familiar voice
    - Warning that posts may be old/new (unknown date)
  - Context injected:
    - Addressee name: `{{ $('Map Lead Data').first().json['firstName, lastName'] }}`
    - LinkedInContext/PersonContext/UniqueAngles from `Analyze LinkedIn Context` output: `{{ $json.message.content.LinkedInContext }}` etc.
  - Includes example assistant messages (few-shot).
- **Connections:**
  - **Input ←** Analyze LinkedIn Context
  - **Output →** Update Row (LinkedIn Data)
- **Edge cases / failures:**
  - Uses `$('Map Lead Data').first()` which assumes a stable item context; in complex multi-item runs, ensure the “first” is still the correct lead. Prefer `$('Map Lead Data').item` when possible to avoid cross-item leakage.
- **Version notes:** `typeVersion: 1.8`

#### Node: Update Row (LinkedIn Data)
- **Type / role:** Google Sheets — writes generated icebreaker/intro/companyType back to the sheet.
- **Configuration (interpreted):**
  - Same pattern as website update:
    - match by `email_final`
    - write `Icebreaker`, `intro`, `companyType`
- **Connections:**
  - **Input ←** Write Email Copy (LinkedIn Context)
  - **Output →** Pass-through (LinkedIn)
- **Edge cases / failures:** Same as website update (matching uniqueness, empty email, API limits).
- **Version notes:** `typeVersion: 4.6`

#### Node: Pass-through (LinkedIn)
- **Type / role:** Set — no assignments; `alwaysOutputData: true`.
- **Connections:**
  - **Input ←** Update Row (LinkedIn Data)
  - **Output →** Merge Loops (input 1)
- **Edge cases / failures:** None meaningful.
- **Version notes:** `typeVersion: 3.4`

---

### Block 7 — Branch Merge & Loop Continuation
**Overview:** Merges the Website and LinkedIn branches into a single continuation path and triggers the next iteration of the SplitInBatches loop.  
**Nodes Involved:**  
- Merge Loops

#### Node: Merge Loops
- **Type / role:** Merge (`n8n-nodes-base.merge`) — recombines branch outputs.
- **Configuration (interpreted):**
  - No explicit merge mode shown in JSON; default merge mode applies (commonly “Append” or “Wait for both” depending on node defaults/version). Given the design (either website OR LinkedIn branch runs), you typically want **Append** so it proceeds with whichever branch produced output.
  - `alwaysOutputData: true` to prevent stalling if one input is empty.
- **Connections:**
  - **Input 0 ←** Pass-through (Website)
  - **Input 1 ←** Pass-through (LinkedIn)
  - **Output →** Loop Over Leads (to continue processing remaining items)
- **Edge cases / failures:**
  - If Merge mode is “Wait for both”, the workflow can deadlock because only one branch executes. Confirm Merge node is set to a non-blocking mode (Append is typical).
- **Version notes:** `typeVersion: 3.2`

---

### Block 8 — Documentation / Annotations (Sticky Notes)
**Overview:** Sticky notes describe configuration requirements and intended behavior. They do not affect execution but are important for reproduction and maintenance.  
**Nodes Involved:**  
- Header Note
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

#### Node: Header Note
- **Type / role:** Sticky Note (`n8n-nodes-base.stickyNote`) — high-level overview and setup requirements.
- **Content (preserved):**
  - “Multi-Channel Cold Email Generator”
  - Requirements: Google Sheets columns; Apify run-sync actor; OpenAI GPT-4 access
  - Setup: replace sheet IDs, add Apify token, update prompts with company name/business type
- **Edge cases:** None (non-executable).

#### Node: Sticky Note
- **Content:** “1. Input Data” requirements: `email_final`, `linkedin_url`, `companyWebsite` (note: workflow actually references `organization_website_url` mapped into `companyWebsite`).
- **Edge cases:** None.

#### Node: Sticky Note1
- **Content:** “2. Scraping Logic” explains LinkedIn scrape first, then fallback to website.
- **Edge cases:** None.

#### Node: Sticky Note2
- **Content:** “3. Website Fallback Branch” explains HTML→Markdown→GPT unique angles.
- **Edge cases:** None.

#### Node: Sticky Note3
- **Content:** “4. LinkedIn Branch” explains aggregating posts + GPT.
- **Edge cases:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Entry point (manual run) | — | Fetch Leads |  |
| Fetch Leads | Google Sheets | Read lead rows (filtered) | When clicking ‘Execute workflow’ | Loop Over Leads | ## 1. Input Data<br>Configure your Google Sheet.<br>**Requirements:**<br>- `email_final`<br>- `linkedin_url`<br>- `companyWebsite` |
| Loop Over Leads | Split In Batches | Iterate through leads | Fetch Leads, Merge Loops | Map Lead Data |  |
| Map Lead Data | Set | Normalize lead fields | Loop Over Leads | Apify LinkedIn Scraper | ## 2. Scraping Logic<br>First, we attempt to scrape LinkedIn posts via Apify.<br><br>**If Posts Found:** We generate personalization based on recent content.<br><br>**If No Posts:** We fallback to scraping the company website directly. |
| Apify LinkedIn Scraper | HTTP Request | Scrape LinkedIn posts (Apify actor) | Map Lead Data | Posts Found? | ## 2. Scraping Logic<br>First, we attempt to scrape LinkedIn posts via Apify.<br><br>**If Posts Found:** We generate personalization based on recent content.<br><br>**If No Posts:** We fallback to scraping the company website directly. |
| Posts Found? | IF | Route to LinkedIn vs website fallback | Apify LinkedIn Scraper | Scrape Company Website; Aggregate Posts | ## 2. Scraping Logic<br>First, we attempt to scrape LinkedIn posts via Apify.<br><br>**If Posts Found:** We generate personalization based on recent content.<br><br>**If No Posts:** We fallback to scraping the company website directly. |
| Scrape Company Website | HTTP Request | Fetch website HTML | Posts Found? (true) | Convert HTML to Markdown | ## 3. Website Fallback Branch<br>If LinkedIn fails, we convert the website HTML to Markdown and use GPT-4 to find unique angles. |
| Convert HTML to Markdown | Markdown | Convert HTML to Markdown text | Scrape Company Website | Analyze Website Context | ## 3. Website Fallback Branch<br>If LinkedIn fails, we convert the website HTML to Markdown and use GPT-4 to find unique angles. |
| Analyze Website Context | OpenAI (LangChain) | Summarize website/person context to JSON | Convert HTML to Markdown | Write Email Copy (Website Context) | ## 3. Website Fallback Branch<br>If LinkedIn fails, we convert the website HTML to Markdown and use GPT-4 to find unique angles. |
| Write Email Copy (Website Context) | OpenAI (LangChain) | Generate icebreaker/intro/companyType JSON | Analyze Website Context | Update Row (Website Data) | ## 3. Website Fallback Branch<br>If LinkedIn fails, we convert the website HTML to Markdown and use GPT-4 to find unique angles. |
| Update Row (Website Data) | Google Sheets | Update lead row with generated copy | Write Email Copy (Website Context) | Pass-through (Website) | ## 3. Website Fallback Branch<br>If LinkedIn fails, we convert the website HTML to Markdown and use GPT-4 to find unique angles. |
| Pass-through (Website) | Set | Normalize/continue flow | Update Row (Website Data) | Merge Loops |  |
| Aggregate Posts | Aggregate | Combine post items into fields | Posts Found? (false) | Analyze LinkedIn Context | ## 4. LinkedIn Branch<br>If posts exist, we aggregate them and use GPT-4 to write an icebreaker referencing specific post content. |
| Analyze LinkedIn Context | OpenAI (LangChain) | Summarize LinkedIn posts/person context to JSON | Aggregate Posts | Write Email Copy (LinkedIn Context) | ## 4. LinkedIn Branch<br>If posts exist, we aggregate them and use GPT-4 to write an icebreaker referencing specific post content. |
| Write Email Copy (LinkedIn Context) | OpenAI (LangChain) | Generate icebreaker/intro/companyType JSON | Analyze LinkedIn Context | Update Row (LinkedIn Data) | ## 4. LinkedIn Branch<br>If posts exist, we aggregate them and use GPT-4 to write an icebreaker referencing specific post content. |
| Update Row (LinkedIn Data) | Google Sheets | Update lead row with generated copy | Write Email Copy (LinkedIn Context) | Pass-through (LinkedIn) | ## 4. LinkedIn Branch<br>If posts exist, we aggregate them and use GPT-4 to write an icebreaker referencing specific post content. |
| Pass-through (LinkedIn) | Set | Normalize/continue flow | Update Row (LinkedIn Data) | Merge Loops |  |
| Merge Loops | Merge | Merge branches and continue batch loop | Pass-through (Website), Pass-through (LinkedIn) | Loop Over Leads |  |
| Header Note | Sticky Note | Global overview & setup instructions | — | — | # 🤖 Multi-Channel Cold Email Generator<br><br>**Overview:**<br>This workflow personalizes cold emails by checking a lead's LinkedIn activity. If they have recent posts, it writes an icebreaker about them. If they don't, it falls back to analyzing their company website.<br><br>**Requirements:**<br>1. **Google Sheets:** Columns for `email_final`, `linkedin_url`, `companyWebsite`.<br>2. **Apify Account:** Using the `run-sync-get-dataset-items` actor.<br>3. **OpenAI API:** Access to GPT-4.<br><br>**Setup Instructions:**<br>1. Open the **Fetch Leads** and **Update Row** nodes. Replace `YOUR_GOOGLE_SHEET_ID` and map your columns.<br>2. Open the **Apify LinkedIn Scraper** node and add your Apify API Token.<br>3. Open the **Write Email Copy** nodes (both branches) and update the System Prompt with your actual Company Name and Business Type. |
| Sticky Note | Sticky Note | Notes: input requirements | — | — | ## 1. Input Data<br>Configure your Google Sheet.<br>**Requirements:**<br>- `email_final`<br>- `linkedin_url`<br>- `companyWebsite` |
| Sticky Note1 | Sticky Note | Notes: scraping logic | — | — | ## 2. Scraping Logic<br>First, we attempt to scrape LinkedIn posts via Apify.<br><br>**If Posts Found:** We generate personalization based on recent content.<br><br>**If No Posts:** We fallback to scraping the company website directly. |
| Sticky Note2 | Sticky Note | Notes: website fallback | — | — | ## 3. Website Fallback Branch<br>If LinkedIn fails, we convert the website HTML to Markdown and use GPT-4 to find unique angles. |
| Sticky Note3 | Sticky Note | Notes: LinkedIn branch | — | — | ## 4. LinkedIn Branch<br>If posts exist, we aggregate them and use GPT-4 to write an icebreaker referencing specific post content. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add node: Google Sheets (Fetch Leads)**
   - Node type: **Google Sheets**
   - Name: `Fetch Leads`
   - Credentials: connect **Google Sheets OAuth2**
   - Operation: **Read/Get Many** (exact label depends on n8n version)
   - Select your **Document** (spreadsheet) and **Sheet** (tab).
   - Configure a filter so you only fetch leads that need enrichment (commonly: `Icebreaker` is empty).
   - Ensure the sheet contains (at minimum):
     - `email_final`
     - `linkedin_url`
     - a website URL column (in this workflow it’s `organization_website_url` which gets mapped to `companyWebsite`)

4. **Connect:** Manual Trigger → Fetch Leads

5. **Add node: Split In Batches**
   - Node type: **Split In Batches**
   - Name: `Loop Over Leads`
   - Set **Batch Size** to something safe (e.g., 1–10) depending on rate limits.
6. **Connect:** Fetch Leads → Loop Over Leads

7. **Add node: Set (Map Lead Data)**
   - Node type: **Set**
   - Name: `Map Lead Data`
   - Add fields (expressions) mapping your sheet columns to:
     - `firstName, lastName` = `{{$json.first_name}}, {{$json.last_name}}`
     - `email` = `{{$json.email_final}}`
     - `city` = `{{$json.city}}`
     - `state` = `{{$json.state}}`
     - `country` = `{{$json.country}}`
     - `companyWebsite` = `{{$json.organization_website_url}}`
     - `companyName` = `{{$json.organization_name}}`
     - `keywords` = `{{$json.keywords}}`
     - `contactLinkedIn` = `{{$json.linkedin_url}}`
     - `companyLinkedIn` = `{{$json.organization_linkedin_url}}`
     - `organizationDescription` = `{{$json.organization_short_description}}`
8. **Connect:** Loop Over Leads (continue output) → Map Lead Data

9. **Add node: HTTP Request (Apify LinkedIn Scraper)**
   - Node type: **HTTP Request**
   - Name: `Apify LinkedIn Scraper`
   - Method: **POST**
   - URL: `https://api.apify.com/v2/acts/A3cAPGpwBEG8RJwse/run-sync-get-dataset-items`
   - Headers: `Authorization: Bearer <YOUR_APIFY_TOKEN>`
   - Body type: **JSON**
   - Body: include the options from the workflow and compute `targetUrls` from `contactLinkedIn` (support comma-separated).
   - Enable retries: e.g., **Retry on fail**, max 5 tries, 5s wait.
10. **Connect:** Map Lead Data → Apify LinkedIn Scraper

11. **Add node: IF (Posts Found?)**
   - Node type: **IF**
   - Name: `Posts Found?`
   - Condition: String equals
     - Left: `{{$json.message}}`
     - Right: `No posts found.`
12. **Connect:** Apify LinkedIn Scraper → Posts Found?

### Website branch (true: no posts)

13. **Add node: HTTP Request (Scrape Company Website)**
   - Name: `Scrape Company Website`
   - Method: GET
   - URL: `{{$('Map Lead Data').item.json.companyWebsite}}`
   - Error handling: set **Continue on fail** (or equivalent) so one bad website doesn’t stop the loop.
14. **Connect:** Posts Found? (true) → Scrape Company Website

15. **Add node: Markdown (Convert HTML to Markdown)**
   - Name: `Convert HTML to Markdown`
   - HTML input: `{{$json.data}}` (or the HTTP node’s body field depending on your HTTP Request settings)
   - Enable continue on fail (optional).
16. **Connect:** Scrape Company Website → Convert HTML to Markdown

17. **Add node: OpenAI (Analyze Website Context)**
   - Node type: **OpenAI (LangChain)**
   - Name: `Analyze Website Context`
   - Credentials: **OpenAI API**
   - Model: `gpt-4-turbo`
   - Temperature: 0.6
   - Enable **JSON output**
   - Prompt: instruct it to output only:
     - `WebsiteContext`, `PersonContext`, `UniqueAngles`
   - Provide website content (truncate to ~5000 chars) + lead metadata from Map Lead Data.
18. **Connect:** Convert HTML to Markdown → Analyze Website Context

19. **Add node: OpenAI (Write Email Copy (Website Context))**
   - Name: `Write Email Copy (Website Context)`
   - Model: `gpt-4`
   - Temperature: 0.7
   - Enable **JSON output**
   - Prompt: generate JSON:
     - `icebreaker`, `intro`, `companyType`
   - Update the prompt placeholders `[YOUR_BUSINESS_TYPE]` and `[YOUR_COMPANY_NAME]` with your real values.
   - Inject context from the previous node’s JSON fields.
20. **Connect:** Analyze Website Context → Write Email Copy (Website Context)

21. **Add node: Google Sheets (Update Row (Website Data))**
   - Name: `Update Row (Website Data)`
   - Credentials: Google Sheets OAuth2
   - Operation: **Update**
   - Match column: `email_final`
   - Set values:
     - `Icebreaker` = generated `icebreaker`
     - `intro` = generated `intro`
     - `companyType` = generated `companyType`
     - `email_final` = original `email_final` from Fetch Leads
   - Enable retry on fail if available.
22. **Connect:** Write Email Copy (Website Context) → Update Row (Website Data)

23. **Add node: Set (Pass-through (Website))**
   - Name: `Pass-through (Website)`
   - No fields; enable **Always Output Data**.
24. **Connect:** Update Row (Website Data) → Pass-through (Website)

### LinkedIn branch (false: posts exist)

25. **Add node: Aggregate (Aggregate Posts)**
   - Name: `Aggregate Posts`
   - Aggregate fields:
     - `type`
     - `content` (output field name: `content`)
     - `repost.content` (output field name: `repostContent`)
26. **Connect:** Posts Found? (false) → Aggregate Posts

27. **Add node: OpenAI (Analyze LinkedIn Context)**
   - Name: `Analyze LinkedIn Context`
   - Model: `gpt-4`
   - Temperature: 0.6
   - Enable **JSON output**
   - Prompt: output only JSON:
     - `LinkedInContext`, `PersonContext`, `UniqueAngles`
   - Inject `content` and `repostContent` + lead metadata from Map Lead Data.
28. **Connect:** Aggregate Posts → Analyze LinkedIn Context

29. **Add node: OpenAI (Write Email Copy (LinkedIn Context))**
   - Name: `Write Email Copy (LinkedIn Context)`
   - Model: `gpt-4`
   - Temperature: 0.7
   - Enable **JSON output**
   - Prompt: same output JSON keys as website branch.
   - Inject:
     - Addressee name from Map Lead Data
     - LinkedInContext/PersonContext/UniqueAngles from Analyze LinkedIn Context
   - Replace your business placeholders.
30. **Connect:** Analyze LinkedIn Context → Write Email Copy (LinkedIn Context)

31. **Add node: Google Sheets (Update Row (LinkedIn Data))**
   - Name: `Update Row (LinkedIn Data)`
   - Same config as website update: match by `email_final`, write `Icebreaker`, `intro`, `companyType`.
32. **Connect:** Write Email Copy (LinkedIn Context) → Update Row (LinkedIn Data)

33. **Add node: Set (Pass-through (LinkedIn))**
   - Name: `Pass-through (LinkedIn)`
   - No fields; **Always Output Data** enabled.
34. **Connect:** Update Row (LinkedIn Data) → Pass-through (LinkedIn)

### Merge + loopback

35. **Add node: Merge (Merge Loops)**
   - Name: `Merge Loops`
   - Set merge mode to a non-blocking option (commonly **Append**) so it doesn’t wait for both branches.
   - Enable **Always Output Data**.
36. **Connect:**
   - Pass-through (Website) → Merge Loops (Input 0)
   - Pass-through (LinkedIn) → Merge Loops (Input 1)

37. **Connect loopback:** Merge Loops → Loop Over Leads  
   This is what advances to the next lead.

38. **Credentials checklist**
   - Google Sheets OAuth2: spreadsheet read + update permissions.
   - OpenAI API credential with access to selected models (`gpt-4`, `gpt-4-turbo`).
   - Apify API token in HTTP header.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Configure Google Sheet columns and mappings; required minimum fields are `email_final`, `linkedin_url`, and a website URL mapped to `companyWebsite`. | From sticky notes and Set/Update mappings |
| Apify actor endpoint used: `run-sync-get-dataset-items` for actor `A3cAPGpwBEG8RJwse`. Add `Authorization: Bearer <token>`. | Apify LinkedIn Scraper node |
| Update both “Write Email Copy” prompts to replace `[YOUR_BUSINESS_TYPE]` and `[YOUR_COMPANY_NAME]`. | Header Note + both email copy nodes |
| Ensure Merge node does not wait for both branches; otherwise the workflow can stall because only one branch runs. | Merge Loops design constraint |