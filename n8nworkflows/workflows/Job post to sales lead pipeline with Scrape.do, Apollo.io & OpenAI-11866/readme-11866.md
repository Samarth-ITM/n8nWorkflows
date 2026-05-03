Job post to sales lead pipeline with Scrape.do, Apollo.io & OpenAI

https://n8nworkflows.xyz/workflows/job-post-to-sales-lead-pipeline-with-scrape-do--apollo-io---openai-11866


# Job post to sales lead pipeline with Scrape.do, Apollo.io & OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Job post to sales lead pipeline with Scrape.do, Apollo.io & OpenAI

**Purpose:**  
This workflow automates sales lead sourcing from **Indeed job listings**. It:
1) Collects job search criteria (manual or form),  
2) Scrapes Indeed results via **Scrape.do**,  
3) Extracts company names and job context,  
4) Enriches organizations + finds decision-makers via **Apollo.io**,  
5) Generates a **personalized LinkedIn connection request** via **OpenAI**,  
6) Saves results to **Google Sheets**.

**Target use cases:**
- Building outbound lead lists based on hiring signals (companies recruiting for relevant roles).
- Quickly identifying technical decision-makers (CTO/VP Eng/Founder) at hiring companies.
- Automating first-touch personalized outreach copy.

### Logical Blocks
**1.1 Input Reception & Search Setup**  
Manual trigger or form submission → normalize parameters → build an Indeed search URL.

**1.2 Job Scraping (Indeed via Scrape.do) & Parsing**  
Fetch Indeed HTML rendered as markdown → parse markdown into structured job/company rows.

**1.3 Company Logging (Google Sheets)**  
Append discovered companies/job context to a “Companies” sheet (acts as log; not true dedupe).

**1.4 Apollo Enrichment: Organization → People**  
Search org by company name → extract org metadata → search for relevant people by titles.

**1.5 Lead Formatting → AI Personalization → Storage**  
Normalize lead fields → generate personalized LinkedIn note → merge message with lead → append to “Leads” sheet.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Search Setup
**Overview:**  
Accepts search criteria either from a manual run or a form submission, then standardizes inputs and constructs the Indeed URL used for scraping.

**Nodes involved:**
- **Manual Trigger**
- **Form Trigger (Optional)**
- **Set Search Parameters**

#### Node: Manual Trigger
- **Type / role:** `manualTrigger` — entry point for interactive testing.
- **Configuration:** No parameters.
- **Inputs/Outputs:** No inputs; outputs a single empty item to start the flow.
- **Connections:**  
  - Output → **Set Search Parameters**
- **Edge cases:** None (only used to start execution).

#### Node: Form Trigger (Optional)
- **Type / role:** `formTrigger` — entry point for end-user configuration via hosted form.
- **Configuration choices:**
  - Form title: “Indeed Job Search Configuration”
  - Fields:
    - “Job Title / Keywords” (required)
    - “Location”
    - “Days Posted (1-30)” (number)
  - Webhook ID: `lead-pipeline-form`
- **Outputs:** Produces a JSON object with keys matching the field labels (e.g., `Job Title / Keywords`).
- **Connections:**  
  - Output → **Set Search Parameters**
- **Version requirements:** Node typeVersion `2.2` (ensure your n8n instance supports Form Trigger).
- **Edge cases / failures:**
  - Empty or missing optional fields handled downstream via fallbacks.
  - If form URL is not reachable (n8n not publicly accessible), external users can’t submit.

#### Node: Set Search Parameters
- **Type / role:** `set` — normalizes incoming fields and builds the Indeed search URL.
- **Configuration choices (interpreted):**
  - Creates:
    - `jobTitle` (string): from form field `Job Title / Keywords` or default `'web scraping'`
    - `location` (string): from form field `Location` or default `'United States'`
    - `daysPosted` (number): from form field `Days Posted (1-30)` or default `14`
    - `indeedUrl` (string): built with URL-encoded title/location and `fromage` (days freshness)
- **Key expressions:**
  - `{{ $json['Job Title / Keywords'] || 'web scraping' }}`
  - `https://www.indeed.com/jobs?q={{ encodeURIComponent(...) }}&l={{ encodeURIComponent(...) }}&fromage={{ ... }}`
- **Connections:**  
  - Input: Manual Trigger OR Form Trigger  
  - Output → **Scrape.do Indeed API**
- **Edge cases / failures:**
  - If the form field names are changed, expressions referencing `Job Title / Keywords` etc. will break.
  - If `daysPosted` is outside Indeed’s accepted range, results may degrade (no validation node is present).

---

### 2.2 Job Scraping (Indeed via Scrape.do) & Parsing
**Overview:**  
Uses Scrape.do to fetch Indeed search results as rendered markdown and converts them into structured job entries with company names.

**Nodes involved:**
- **Scrape.do Indeed API**
- **Parse Indeed Jobs**

#### Node: Scrape.do Indeed API
- **Type / role:** `httpRequest` — calls Scrape.do to scrape Indeed.
- **Configuration choices:**
  - URL: `https://api.scrape.do`
  - Method: GET (implicit)
  - Auth: **HTTP Query Auth** (generic credential)
  - Query parameters:
    - `url` = `{{$json.indeedUrl}}` (Indeed search URL)
    - `super=true` (Scrape.do enhanced mode)
    - `geoCode=us`
    - `render=true` (enable JS rendering)
    - `blockResources=true` (performance)
    - `device=mobile` (mobile UA)
    - `output=markdown` (important for downstream parsing)
  - Response: **text**
  - Timeout: 60s
- **Connections:**  
  - Input ← **Set Search Parameters**  
  - Output → **Parse Indeed Jobs**
- **Edge cases / failures:**
  - Credential missing/invalid → 401/403 from Scrape.do.
  - Indeed anti-bot / degraded results → markdown may not contain expected patterns.
  - Large pages or slow render → timeout (60s).
  - Output format change (markdown structure differs) will break parsing.

#### Node: Parse Indeed Jobs
- **Type / role:** `code` — parses markdown text into job records; deduplicates by company.
- **Core logic (interpreted):**
  - Reads markdown from `input.data`, `input.body`, or stringifies fallback.
  - Iterates line-by-line:
    - Detects job headers like: `## [Title](...viewjob?jk=JOBID...)`
    - Builds `currentJob` object with fields:
      - `jobTitle`, `jobUrl`, `jobId`, `companyName`, `location`, `salary`, `jobType`, `source='Indeed'`, `dateFound=today`
    - Uses heuristics to:
      - Stop parsing when certain “stop markers” appear.
      - Identify salary lines, job type lines, location lines.
      - Identify company name within first ~5 meaningful lines after title.
  - Filters out invalid jobs and deduplicates by lowercase companyName.
  - Returns one n8n item per unique company/job.
- **Connections:**  
  - Input ← **Scrape.do Indeed API**  
  - Output → **Add New Company**
- **Edge cases / failures:**
  - If Scrape.do returns HTML/error text, parsing may return 0 items (workflow ends silently).
  - Heuristic mis-detection can produce:
    - Missing `companyName` (then job is discarded)
    - Wrong companyName (then Apollo search may fail or find wrong org)
  - Deduplication is only by company name (different roles at same company collapse to one).

---

### 2.3 Company Logging (Google Sheets)
**Overview:**  
Stores each parsed job/company row in Google Sheets as a “Companies” log. Despite the sticky note claim, this workflow does not actually check for duplicates in Sheets before appending.

**Nodes involved:**
- **Add New Company**

#### Node: Add New Company
- **Type / role:** `googleSheets` — append company/job rows to a spreadsheet tab.
- **Configuration choices:**
  - Operation: **Append**
  - Document: Google Sheet named “New Company” (ID is configured in node)
  - Sheet/tab: “Sheet1” (`gid=0`)  
    - Sticky note suggests a tab named `Companies`, but node points to `Sheet1`.
  - Mapped columns (written):
    - `companyName`, `jobTitle`, `jobUrl`, `location`, `salary`, `source`, `dateFound`
- **Connections:**  
  - Input ← **Parse Indeed Jobs**  
  - Output → **Apollo Organization Search**
- **Edge cases / failures:**
  - Google auth missing/expired → 401/403.
  - Sheet schema mismatch (column names differ) → data may go to wrong columns or fail.
  - Append-only behavior → duplicates accumulate unless handled externally.
- **Version requirements:** googleSheets typeVersion `4.7`.

---

### 2.4 Apollo Enrichment: Organization → People
**Overview:**  
Uses Apollo to find the best-matching organization for each company and then searches for up to three decision-makers by job titles.

**Nodes involved:**
- **Apollo Organization Search**
- **Extract Apollo Org Data**
- **Apollo People Search**

#### Node: Apollo Organization Search
- **Type / role:** `httpRequest` — Apollo Organizations Search API.
- **Configuration choices:**
  - Endpoint: `POST https://api.apollo.io/v1/organizations/search`
  - Auth: **HTTP Header Auth** (generic credential; typically `x-api-key: ...`)
  - Header: `Content-Type: application/json`
  - JSON body:
    - `q_organization_name`: `{{$json.companyName}}`
    - `page: 1`, `per_page: 1` (take best match only)
  - Response: full response enabled
  - Timeout: 30s
- **Connections:**  
  - Input ← **Add New Company**  
  - Output → **Extract Apollo Org Data**
- **Edge cases / failures:**
  - `companyName` empty → Apollo may error or return empty results (noted in troubleshooting sticky).
  - Wrong org match due to ambiguous company names.
  - Rate limiting (429) — workflow suggests adding a Wait node.
  - Credential header misconfigured (common).

#### Node: Extract Apollo Org Data
- **Type / role:** `code` — merges Apollo org response with original company/job context.
- **Core logic (interpreted):**
  - For each incoming item:
    - Read `response.body.organizations[0]` as org.
    - Pull “originalData” by referencing **Add New Company** at the same item index:  
      `$('Add New Company').item(index).json`
    - Output a combined object:
      - Original: `companyName, jobTitle, jobUrl, location, dateFound`
      - Apollo org fields:
        - `linkedinUrl`, `organizationId`, `apolloOrganizationName`, `websiteUrl`, `industry`,
          `employeeCount`, `foundedYear`, `city/state/country`, `description`
      - Flags: `apolloEnriched` (boolean), `enrichmentTimestamp`
- **Connections:**  
  - Input ← **Apollo Organization Search**  
  - Output → **Apollo People Search**
- **Edge cases / failures:**
  - If **Add New Company** output count mismatches Apollo response count, index-based lookup can misalign.
  - If Apollo response shape changes, `body.organizations` might not exist → outputs null fields.
  - `organizationId` null will later degrade people search.

#### Node: Apollo People Search
- **Type / role:** `httpRequest` — Apollo Mixed People Search API.
- **Configuration choices:**
  - Endpoint: `POST https://api.apollo.io/v1/mixed_people/search`
  - Auth: HTTP Header Auth (same as org search)
  - Header: `Content-Type: application/json`
  - JSON body:
    - `organization_ids`: `[ "{{$json.organizationId}}" ]`
    - `person_titles`: CTO/VP Eng/Head of Eng/Eng Manager/Technical Director/CEO/Founder
    - `page: 1`, `per_page: 3`
  - Response: full response enabled
  - Timeout: 30s
- **Connections:**  
  - Input ← **Extract Apollo Org Data**  
  - Output → **Format Leads**
- **Edge cases / failures:**
  - If `organizationId` is null/empty, Apollo may return empty people.
  - Rate limits (429), timeouts, or empty result sets are common.
  - Title list may miss relevant decision makers (e.g., “Co-founder”, “Director of Engineering”).

---

### 2.5 Lead Formatting → AI Personalization → Storage
**Overview:**  
Transforms Apollo people results into lead rows, asks OpenAI to generate a short personalized LinkedIn connection note, then saves final leads to Google Sheets.

**Nodes involved:**
- **Format Leads**
- **Generate Personalized Message**
- **Merge Lead + Message**
- **Save Leads to Sheet**

#### Node: Format Leads
- **Type / role:** `code` — converts Apollo people response into normalized lead items (one per person).
- **Core logic (interpreted):**
  - Reads `body.people` from each Apollo response.
  - Fetches company context by index from **Extract Apollo Org Data**:  
    `$('Extract Apollo Org Data').item(index).json`
  - Emits one item per person with fields:
    - Person: `firstName,lastName,fullName,title,email,phone,linkedinUrl,apolloPersonId`
    - Company: `companyName,companyWebsite,companyLinkedIn,industry,country,city`
    - Job context: `jobTitle,jobUrl`
    - Pipeline fields: `status='New'`, `dateAdded=today`, `source='Indeed + Apollo'`
- **Connections:**  
  - Input ← **Apollo People Search**  
  - Output → **Generate Personalized Message**
- **Edge cases / failures:**
  - If `people` is empty → returns no items (downstream won’t run).
  - Index alignment risk (same as above).
  - Some Apollo fields may be missing; code defaults to empty strings.

#### Node: Generate Personalized Message
- **Type / role:** `openAi` — generates a LinkedIn connection request message.
- **Configuration choices:**
  - Resource: Chat
  - System instruction: outreach specialist; <300 chars; specific reason; no generic templates; English.
  - User prompt includes:
    - `fullName`, `title`, `companyName`, `industry`, and hiring context `jobTitle`
    - Explicit mention: “expanding their data/scraping team” + “share insights about web scraping solutions”
  - Max tokens: 150, temperature 0.7
- **Connections:**  
  - Input ← **Format Leads**  
  - Output → **Merge Lead + Message**
- **Version requirements:** openAi node typeVersion `1.1` (ensure compatible with your n8n).
- **Edge cases / failures:**
  - Missing OpenAI credential or model availability issues.
  - Output shape can vary by node version; handled downstream with multiple fallbacks.
  - Message may exceed 300 chars despite instruction (no hard enforcement node).

#### Node: Merge Lead + Message
- **Type / role:** `code` — merges OpenAI response text back into the corresponding lead item.
- **Core logic (interpreted):**
  - Extracts message from possible response shapes:
    - `message.content`
    - `choices[0].message.content`
    - raw string fallback
  - Fetches lead by index from **Format Leads**:  
    `$('Format Leads').item(index).json`
  - Outputs final lead fields +:
    - `personalizedMessage`
    - `messageGeneratedAt` (ISO timestamp)
- **Connections:**  
  - Input ← **Generate Personalized Message**  
  - Output → **Save Leads to Sheet**
- **Edge cases / failures:**
  - Index mismatch if OpenAI returns fewer/more items (rare but possible with errors/retries).
  - Empty message if OpenAI returns an unexpected structure.

#### Node: Save Leads to Sheet
- **Type / role:** `googleSheets` — appends finalized leads to a “Leads” tab.
- **Configuration choices:**
  - Operation: Append
  - Document: “read existing” (spreadsheet configured in node)
  - Sheet/tab: `Leads`
  - Columns mapped:
    - First Name, Last Name, Title, Company, LinkedIn URL, Country, Industry,
      Date Added, Source, Personalized Message
- **Connections:**  
  - Input ← **Merge Lead + Message**
- **Edge cases / failures:**
  - Google Sheets auth/scope issues.
  - Column headers must match exactly (e.g., “LinkedIn URL”).
  - Appending duplicates (no dedupe).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 🎯 Input Options | stickyNote | Documentation / usage notes | — | — | ## How it works… Setup steps… (credentials, create tabs `Companies` and `Leads`, update “Set Search Parameters”, run manually or via form) |
| 🔍 Scrape.do Details | stickyNote | Documentation for scraping block | — | — | ## 1. Job Scraping Accepts search criteria… via Scrape.do… parses markdown |
| 🏢 Apollo Org Search | stickyNote | Documentation for enrichment block | — | — | ## 2. Data Enrichment Logs companies… uses Apollo.io… find key decision-makers |
| 👥 Apollo People | stickyNote | Documentation for personalization/output block | — | — | ## 3. Personalization & Output Formats lead data, OpenAI message, saves to Sheets |
| 🔧 Troubleshooting | stickyNote | Troubleshooting tips | — | — | Apollo error: companyName may be empty; No credentials; Empty results; Rate limit: add Wait node; Test each node individually |
| Manual Trigger | manualTrigger | Manual entry point | — | Set Search Parameters | ## How it works… Setup steps… |
| Form Trigger (Optional) | formTrigger | Form-based entry point | — | Set Search Parameters | ## How it works… Setup steps… |
| Set Search Parameters | set | Normalize inputs + build Indeed URL | Manual Trigger; Form Trigger (Optional) | Scrape.do Indeed API | ## How it works… Setup steps… |
| Scrape.do Indeed API | httpRequest | Scrape Indeed results via Scrape.do | Set Search Parameters | Parse Indeed Jobs | ## 1. Job Scraping… |
| Parse Indeed Jobs | code | Parse markdown into job/company items | Scrape.do Indeed API | Add New Company | ## 1. Job Scraping… |
| Add New Company | googleSheets | Append company/job context into Sheets | Parse Indeed Jobs | Apollo Organization Search | ## 2. Data Enrichment… |
| Apollo Organization Search | httpRequest | Search org by company name (Apollo) | Add New Company | Extract Apollo Org Data | ## 2. Data Enrichment… |
| Extract Apollo Org Data | code | Merge org enrichment with original job context | Apollo Organization Search | Apollo People Search | ## 2. Data Enrichment… |
| Apollo People Search | httpRequest | Find decision-makers by titles (Apollo) | Extract Apollo Org Data | Format Leads | ## 2. Data Enrichment… |
| Format Leads | code | Flatten people results into lead rows | Apollo People Search | Generate Personalized Message | ## 3. Personalization & Output… |
| Generate Personalized Message | openAi | Create LinkedIn connection message | Format Leads | Merge Lead + Message | ## 3. Personalization & Output… |
| Merge Lead + Message | code | Combine OpenAI message with lead data | Generate Personalized Message | Save Leads to Sheet | ## 3. Personalization & Output… |
| Save Leads to Sheet | googleSheets | Append finalized leads to “Leads” sheet | Merge Lead + Message | — | ## 3. Personalization & Output… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n named:  
   “Job post to sales lead pipeline with Scrape.do, Apollo.io & OpenAI”.

2) **Add entry points**
   1. Add **Manual Trigger**.
   2. Add **Form Trigger** (optional):
      - Title: “Indeed Job Search Configuration”
      - Description: “Enter your search criteria to find job postings and generate leads.”
      - Fields:
        - Job Title / Keywords (required, text)
        - Location (text)
        - Days Posted (1-30) (number)

3) **Add “Set Search Parameters” (Set node)**
   - Add fields:
     - `jobTitle` (String): `{{ $json['Job Title / Keywords'] || 'web scraping' }}`
     - `location` (String): `{{ $json['Location'] || 'United States' }}`
     - `daysPosted` (Number): `{{ $json['Days Posted (1-30)'] || 14 }}`
     - `indeedUrl` (String):
       ```
       https://www.indeed.com/jobs?q={{ encodeURIComponent($json['Job Title / Keywords'] || 'web scraping') }}&l={{ encodeURIComponent($json['Location'] || 'United States') }}&fromage={{ $json['Days Posted (1-30)'] || 14 }}
       ```
   - Connect:
     - Manual Trigger → Set Search Parameters
     - Form Trigger (Optional) → Set Search Parameters

4) **Add “Scrape.do Indeed API” (HTTP Request)**
   - Method: GET
   - URL: `https://api.scrape.do`
   - Authentication: **Generic Credential → HTTP Query Auth**
     - Configure credential with your Scrape.do API key as required by your Scrape.do plan (query-based auth).
   - Query parameters:
     - `url` = `{{ $json.indeedUrl }}`
     - `super` = `true`
     - `geoCode` = `us`
     - `render` = `true`
     - `blockResources` = `true`
     - `device` = `mobile`
     - `output` = `markdown`
   - Response format: **Text**
   - Timeout: 60000 ms
   - Connect: Set Search Parameters → Scrape.do Indeed API

5) **Add “Parse Indeed Jobs” (Code node)**
   - Paste the parsing logic (adapted from the workflow) that:
     - Reads returned markdown
     - Extracts `jobTitle`, `jobUrl`, `jobId`, `companyName`, `location`, `salary`, `jobType`
     - Filters + deduplicates by company name
     - Outputs one item per company/job
   - Connect: Scrape.do Indeed API → Parse Indeed Jobs

6) **Prepare Google Sheets**
   - Create (or choose) a spreadsheet for company logging. Ensure headers exist for:
     - `companyName, jobTitle, jobUrl, location, salary, source, dateFound`
   - Create (or choose) a second spreadsheet (or same) for leads with headers:
     - `First Name, Last Name, Title, Company, LinkedIn URL, Country, Industry, Date Added, Source, Personalized Message`
   - Note: The sticky note suggests tabs named `Companies` and `Leads`, but the provided workflow actually writes the company log to a tab called `Sheet1`. Decide one convention and align node configuration accordingly.

7) **Add “Add New Company” (Google Sheets node)**
   - Credentials: Google Sheets OAuth2 (configure in n8n Credentials).
   - Operation: Append
   - Select the spreadsheet and the “Companies” tab (or `Sheet1`, but be consistent).
   - Map columns from parsed job item:
     - `companyName={{$json.companyName}}`, `jobTitle`, `jobUrl`, `location`, `salary`, `source`, `dateFound`
   - Connect: Parse Indeed Jobs → Add New Company

8) **Add Apollo credentials**
   - In n8n Credentials, create **HTTP Header Auth** credential for Apollo.
   - Configure header (typical): `x-api-key: <YOUR_APOLLO_KEY>`  
     (Exact header name can vary; match Apollo’s current documentation/account settings.)

9) **Add “Apollo Organization Search” (HTTP Request)**
   - Method: POST
   - URL: `https://api.apollo.io/v1/organizations/search`
   - Auth: Generic Credential → HTTP Header Auth (Apollo)
   - Headers: `Content-Type: application/json`
   - Body (JSON):
     - `q_organization_name: {{$json.companyName}}`
     - `page: 1`
     - `per_page: 1`
   - Connect: Add New Company → Apollo Organization Search

10) **Add “Extract Apollo Org Data” (Code node)**
   - Implement code that:
     - Reads `body.organizations[0]`
     - Merges it with the original company/job context
     - Outputs `organizationId`, `linkedinUrl`, `websiteUrl`, `industry`, etc.
     - (Optionally) references “Add New Company” by item index to preserve job context.
   - Connect: Apollo Organization Search → Extract Apollo Org Data

11) **Add “Apollo People Search” (HTTP Request)**
   - Method: POST
   - URL: `https://api.apollo.io/v1/mixed_people/search`
   - Auth: same Apollo HTTP Header Auth
   - Header: `Content-Type: application/json`
   - Body (JSON):
     - `organization_ids: ["{{$json.organizationId}}"]`
     - `person_titles`: list of decision-maker titles (CTO, VP Eng, Founder, etc.)
     - `page: 1`, `per_page: 3`
   - Connect: Extract Apollo Org Data → Apollo People Search

12) **Add “Format Leads” (Code node)**
   - Convert Apollo `people` into individual lead items.
   - Merge in company/job context from “Extract Apollo Org Data”.
   - Connect: Apollo People Search → Format Leads

13) **Add OpenAI credentials**
   - Create OpenAI credential in n8n (API key).
   - Ensure your n8n OpenAI node is installed/enabled.

14) **Add “Generate Personalized Message” (OpenAI node)**
   - Resource: Chat
   - System message: outreach specialist, <300 chars, non-generic, English.
   - User message: include `fullName`, `title`, `companyName`, `industry`, and “They are hiring for {{jobTitle}}”.
   - Settings: temperature 0.7, max tokens ~150.
   - Connect: Format Leads → Generate Personalized Message

15) **Add “Merge Lead + Message” (Code node)**
   - Extract the generated message from the OpenAI response (handle possible response shapes).
   - Merge into the lead object as `personalizedMessage`.
   - Connect: Generate Personalized Message → Merge Lead + Message

16) **Add “Save Leads to Sheet” (Google Sheets node)**
   - Operation: Append
   - Spreadsheet: your “Leads” spreadsheet/tab
   - Map:
     - First Name, Last Name, Title, Company, LinkedIn URL, Country, Industry,
       Date Added, Source, Personalized Message
   - Connect: Merge Lead + Message → Save Leads to Sheet

17) **(Optional hardening)**
   - Add an **IF** node after parsing to stop if no jobs found.
   - Add a **Wait** node before Apollo calls to reduce rate-limit risk.
   - Add dedupe checks (Sheets lookup or n8n Data Store) before appending.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates the lead sourcing process by scraping job listings from Indeed using Scrape.do… uses OpenAI… saves all qualified leads to Google Sheets.” | Sticky note: “🎯 Input Options” |
| Setup steps: configure Scrape.do, Apollo.io, OpenAI, Google Sheets credentials; create sheet with tabs `Companies` and `Leads`; update “Set Search Parameters”; execute workflow or use form URL. | Sticky note: “🎯 Input Options” |
| Troubleshooting: Apollo error (companyName may be empty), select credentials in nodes, check Scrape.do output, add Wait node for rate limits, test nodes individually. | Sticky note: “🔧 Troubleshooting” |