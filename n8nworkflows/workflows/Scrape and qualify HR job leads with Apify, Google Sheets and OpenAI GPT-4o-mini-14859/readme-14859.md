Scrape and qualify HR job leads with Apify, Google Sheets and OpenAI GPT-4o-mini

https://n8nworkflows.xyz/workflows/scrape-and-qualify-hr-job-leads-with-apify--google-sheets-and-openai-gpt-4o-mini-14859


# Scrape and qualify HR job leads with Apify, Google Sheets and OpenAI GPT-4o-mini

Now I have a thorough understanding. Let me write the complete document.

## 1. Workflow Overview

**Purpose:** This workflow automates HR lead generation for a manpower recruitment agency. It captures job search parameters from a web form, scrapes job listings via the Apify "All Jobs Scraper" actor, deduplicates results against a Google Sheets ledger, enriches missing company names via Firecrawl, scores each job using OpenAI GPT-4o-mini to determine lead qualification, and writes both raw jobs and qualified leads back to Google Sheets.

**Target use cases:**
- Recruiters who need to find and qualify job postings across multiple roles and countries in bulk
- Agencies targeting GCC (UAE, Saudi Arabia, Qatar, Kuwait, Oman, Bahrain) and select Asian markets for hospitality, logistics, healthcare, oil & gas, and construction manpower supply

**Logical blocks:**

| Block | Name | Function |
|-------|------|----------|
| 1 | Input Reception & Parsing | Captures form input, parses comma-separated roles/locations into a cartesian product of search objects |
| 2 | Search Iteration & Job Scraping | Loops over each role×location combination, calls Apify synchronously to scrape job listings |
| 3 | Flatten, Deduplicate & Save | Flattens scraped arrays, generates deterministic hashes, reads existing jobs from Sheets, filters duplicates, appends new unique jobs |
| 4 | Job Processing Loop & Company Enrichment | Iterates each saved job; if `company_name` is missing, scrapes the job URL with Firecrawl to extract `ogDescription` as a fallback company name, then updates the sheet |
| 5 | AI Scoring & Lead Qualification | Sends job context to GPT-4o-mini for scoring (1–10), industry classification, country extraction, and ACCEPT/REJECT decision; writes qualified (status=QUALIFIED) or not-qualified records back to Sheets, then cycles to the next job |

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Parsing

**Overview:** A public n8n form captures the user's job search criteria (roles, locations, max results, posting recency). A Code node splits the comma-separated inputs and produces a cartesian product of every role × location pair, each carrying the shared max-results and posted-since parameters.

**Nodes Involved:** When Job Form Submitted, Parse Form Input Arrays

#### When Job Form Submitted
- **Type:** `n8n-nodes-base.formTrigger` (v2.5)
- **Technical role:** Webhook-based entry point; renders a form UI and emits a single item containing all submitted fields.
- **Configuration:**
  - Form title: "Job Search Configuration"
  - Form description: "Configure job search parameters for lead generation"
  - Fields:
    - **Job Roles** (text, required, placeholder: "Chef, Nurse, Driver, Welder")
    - **Locations** (text, required, placeholder: "Kuwait, UAE, Germany")
    - **Max Results Per Search** (number, placeholder: "Minimum 10")
    - **Posted Since** (dropdown: 24 hours / 3 days / 1 week / 2 weeks / 1 month)
- **Key expressions/variables:** None — outputs raw form fields.
- **Input:** None (trigger).
- **Output:** One item with fields `Job Roles`, `Locations`, `Max Results Per Search`, `Posted Since` → connects to **Parse Form Input Arrays**.
- **Edge cases:**
  - If the user enters an empty string for Job Roles or Locations, the downstream split will produce zero combinations and the workflow will stall silently.
  - The dropdown for "Posted Since" has no default; if omitted, the Code node falls back to "1 week".

#### Parse Form Input Arrays
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical role:** Transforms a single form submission into an array of search objects (one per role × location combination).
- **Configuration:**
  - JavaScript code that:
    1. Splits `Job Roles` by comma, trims whitespace, filters empty strings.
    2. Splits `Locations` by comma, trims whitespace, filters empty strings.
    3. Parses `Max Results Per Search` as integer, defaulting to 10.
    4. Reads `Posted Since`, defaulting to "1 week".
    5. Builds the cartesian product: for each role, for each location, emits `{ job_role, location, max_results, posted_since }`.
- **Key expressions/variables:** `$input.first().json['Job Roles']`, `$input.first().json['Locations']`, `parseInt(...)`.
- **Input:** Single item from the form trigger.
- **Output:** N items (one per role×location) → connects to **Loop Over Search Batches**.
- **Edge cases:**
  - Extra commas (e.g., "Chef,,Nurse") produce empty entries that are filtered out.
  - Non-numeric "Max Results" values fall back to 10 via `|| 10`.

---

### Block 2 — Search Iteration & Job Scraping

**Overview:** A Split In Batches node iterates through each search combination one at a time. For each combination, an HTTP Request calls the Apify "All Jobs Scraper" actor synchronously, waits up to 5 minutes, and returns the scraped job dataset.

**Nodes Involved:** Loop Over Search Batches, Submit Job Scrape Request

#### Loop Over Search Batches
- **Type:** `n8n-nodes-base.splitInBatches` (v3)
- **Technical role:** Sequential iterator — emits one search object at a time (output 1), and fires the "done" signal (output 0) when all search objects have been processed.
- **Configuration:**
  - `reset: false` — the batch state is not reset on re-entry, which matters because this node is re-entered from the job-batch loop's "done" output.
- **Key expressions/variables:** None.
- **Input:** Array of search objects from **Parse Form Input Arrays**.
- **Output:**
  - Output 0 (done): no connection — workflow terminates naturally after all searches complete.
  - Output 1 (each item): → **Submit Job Scrape Request**.
- **Edge cases:**
  - If only one search combination exists, the loop processes it once and immediately exits.
  - Because `reset: false`, if the workflow is re-triggered while a previous execution is still running, batch state may carry over; in practice n8n isolates executions.

#### Submit Job Scrape Request
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical role:** Sends a POST request to the Apify API to run the `agentx/all-jobs-scraper` actor synchronously and return the dataset items directly.
- **Configuration:**
  - URL: `https://api.apify.com/v2/acts/agentx~all-jobs-scraper/run-sync-get-dataset-items`
  - Method: POST
  - Body (JSON):
    - `keyword` → `{{ $json.job_role }}`
    - `country` → `{{ $json.location }}`
    - `max_results` → `{{ $json.max_results }}`
    - `posted_since` → `{{ $json.posted_since }}`
  - Timeout: 300 000 ms (5 minutes)
  - Authentication: Generic credential type → HTTP Header Auth (expects an Apify API token in a header).
- **Key expressions/variables:** `$json.job_role`, `$json.location`, `$json.max_results`, `$json.posted_since`.
- **Input:** Single search object from the batch loop.
- **Output:** Array of job objects from Apify → connects to **Flatten Job Results and ID**.
- **Credentials required:** An `httpHeaderAuth` credential containing the Apify API token (typically as `Authorization: Bearer <APIFY_TOKEN>`). **Note:** The credentials object in the template JSON is empty — this must be configured manually before the workflow can run.
- **Edge cases:**
  - Apify actor timeout exceeding 5 minutes → n8n throws a timeout error.
  - Apify returning an error response (rate limit, invalid API key, actor failure) → the HTTP Request node will error and stop the workflow unless error handling is added.
  - The actor returns dataset items directly; if the actor returns zero results, the downstream Flatten node receives an empty array.

---

### Block 3 — Flatten, Deduplicate & Save

**Overview:** The raw scraped results are flattened into a flat list, enriched with deterministic hashes and unique IDs, compared against existing jobs already stored in Google Sheets, and only genuinely new jobs are appended with status "NEW".

**Nodes Involved:** Flatten Job Results and ID, Read Jobs from Sheets, Identify Duplicate Jobs, Append Raw Jobs to Sheets

#### Flatten Job Results and ID
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical role:** Normalizes the Apify response (which may be an array per item), adds a `unique_id`, a deterministic `job_hash` for deduplication, and annotates each job with its search context.
- **Configuration:**
  - JavaScript code that:
    1. Flattens all input items — if an item's json is an array, it spreads it; otherwise wraps in an array.
    2. Retrieves the current search context from `$('Loop Over Search Batches').item.json` (the role and location that produced this batch).
    3. Computes a `simpleHash` (a basic 32-bit hash converted to base-36) from `job_url` if available, or from `company_name|job_title|job_location` as a fallback.
    4. Produces a compound `job_hash` by hashing the input forward and reversed (to reduce collision probability).
    5. Adds `unique_id` (timestamp + index + random string), `job_hash`, `lead_source: 'job_board'`, `search_role`, `search_location`.
- **Key expressions/variables:** `$('Loop Over Search Batches').item.json`, `simpleHash()`, `job_url`, `company_name`, `job_title`/`designation`, `job_location`/`location`.
- **Input:** Items from **Submit Job Scrape Request** (Apify results).
- **Output:** Flattened items with enrichment → connects to **Read Jobs from Sheets**.
- **Edge cases:**
  - If Apify returns no results, the `flatMap` produces an empty array and the pipeline naturally stops (no items flow forward).
  - Hash collisions are possible with the simple hash function but extremely unlikely for typical job listing data; the compound forward+reverse approach mitigates this further.
  - `job_url` field name depends on what the Apify actor returns; the code checks `job.platform_url` mapping in the downstream sheet write.

#### Read Jobs from Sheets
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Technical role:** Reads all existing rows from the dedup/reference sheet to enable duplicate detection.
- **Configuration:**
  - Operation: Read (default)
  - Document ID: `1hex1u1cpI5sMx8I45ktgpZ4VhXReQA4EZdoCTVEc1bI`
  - Sheet Name (by ID): `1451193517` (a specific tab within the spreadsheet, likely a "Jobs" or "All Jobs" sheet used for dedup lookup)
  - Output formatting: dates as formatted strings, general as unformatted values
  - `alwaysOutputData: true` — ensures the node emits an empty item even if the sheet is empty (preventing downstream "no data" errors)
- **Key expressions/variables:** None.
- **Input:** Items from **Flatten Job Results and ID**.
- **Output:** All existing rows from the sheet → connects to **Identify Duplicate Jobs**.
- **Credentials required:** Google Sheets OAuth2 API credential.
- **Edge cases:**
  - Empty sheet → node emits one empty item (due to `alwaysOutputData`); the dedup code handles this gracefully because `row.json.job_hash` will be falsy.
  - Permission errors → node fails; ensure the Google service account or OAuth user has read access to the spreadsheet.
  - Very large sheets → reading all rows may be slow or hit Google API limits.

#### Identify Duplicate Jobs
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical role:** Compares the `job_hash` of each newly flattened job against the set of hashes already present in the sheet, and filters out any duplicates.
- **Configuration:**
  - JavaScript code that:
    1. Collects all `job_hash` values from the existing sheet rows into a `Set`.
    2. Filters the flattened jobs, keeping only those whose `job_hash` is not in the set.
    3. Returns an empty array if all jobs are duplicates (pipeline stops naturally).
- **Key expressions/variables:** `$('Flatten Job Results and ID').all()`, `$('Read Jobs from Sheets').all()`, `existingHashes` (Set), `job_hash`.
- **Input:** Items from **Read Jobs from Sheets** (but references both the flattened jobs and the sheet rows).
- **Output:** Only unique (non-duplicate) jobs → connects to **Append Raw Jobs to Sheets**.
- **Edge cases:**
  - If all scraped jobs are duplicates, the node returns `[]` and the workflow halts for this search batch (no items reach the Append node).
  - If the sheet has no `job_hash` column, all existing rows are ignored and all new jobs pass through (potential false positives — no duplicates are caught).

#### Append Raw Jobs to Sheets
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Technical role:** Appends new (deduplicated) job records to the main jobs sheet with status "NEW".
- **Configuration:**
  - Operation: Append (`useAppend: true`)
  - Document ID: `1hex1u1cpI5sMx8I45ktgpZ4VhXReQA4EZdoCTVEc1bI`
  - Sheet Name (by ID): `1726393267` (the main jobs/leads tab)
  - Column mapping (defined below schema):
    - `status` → hardcoded "NEW"
    - `job_url` → `{{ $json.platform_url }}` (maps Apify's `platform_url` to `job_url`)
    - `job_type` → `{{ $json.job_type }}`
    - `job_title` → `{{ $json.title }}` (maps Apify's `title` to `job_title`)
    - `unique_id` → `{{ $json.unique_id }}`
    - `created_at` → `{{ $now }}`
    - `job_source` → `{{ $json.platform }}`
    - `company_name` → `{{ $json.company_name }}`
    - `job_location` → `{{ $json.location }}` (maps Apify's `location` to `job_location`)
    - `original_json` → `{{ JSON.stringify($json) }}`
    - `job_description` → `{{ $json.description }}`
    - `job_posted_date` → `{{ $json.posted_date }}`
  - Schema includes additional columns (company_id, job_category, qualified, qualification_reason) that are not populated at this stage — they remain empty for later enrichment.
- **Key expressions/variables:** See column mapping above.
- **Input:** Unique jobs from **Identify Duplicate Jobs**.
- **Output:** Same items (after write) → connects to **Loop Over Job Batches**.
- **Credentials required:** Google Sheets OAuth2 API credential (same as Read node).
- **Edge cases:**
  - Google Sheets API rate limits on rapid appends (one row per item) when many jobs are scraped.
  - If `platform_url`, `title`, or `location` fields are named differently by the Apify actor, those columns will be empty.
  - The `JSON.stringify($json)` for `original_json` could produce very large cells if job descriptions are long; Google Sheets has a 50,000 character cell limit.

---

### Block 4 — Job Processing Loop & Company Enrichment

**Overview:** A second Split In Batches node iterates through each newly saved job one at a time. An IF node checks whether `company_name` is present. If missing, Firecrawl scrapes the job URL to extract `ogDescription` as a fallback company name, updates the sheet, and then the job flows to the AI scoring block. If the company name is already present, the job goes directly to AI scoring.

**Nodes Involved:** Loop Over Job Batches, Check Company Name Presence, Detect Missing Company Names, Append Jobs with Company Names

#### Loop Over Job Batches
- **Type:** `n8n-nodes-base.splitInBatches` (v3)
- **Technical role:** Sequential iterator for individual jobs; re-enters the search batch loop once all jobs in the current batch are done.
- **Configuration:**
  - `reset: false` — batch state persists across re-entries.
- **Key expressions/variables:** None.
- **Input:** Items from **Append Raw Jobs to Sheets** (and from **Cycle Job Processing** on each iteration).
- **Output:**
  - Output 0 (done): → **Loop Over Search Batches** (processes the next search combination).
  - Output 1 (each item): → **Check Company Name Presence**.
- **Edge cases:**
  - If no unique jobs were appended (all were duplicates), this node receives zero items; the "done" output fires immediately and the search batch loop advances.
  - The "done" output re-enters **Loop Over Search Batches** at input 0, which in splitInBatches v3 is the standard input that triggers the next batch.

#### Check Company Name Presence
- **Type:** `n8n-nodes-base.if` (v2.3)
- **Technical role:** Routes jobs based on whether `company_name` is empty/undefined.
- **Configuration:**
  - Combinator: AND
  - Condition: operator `empty` (singleValue) on `leftValue: "=[undefined]"`
  - Loose type validation enabled
- **Key expressions/variables:** The `leftValue` is set to `=[undefined]` — **this is a potential configuration issue.** It appears the intent is to check whether `$json.company_name` is empty or undefined, but the expression as written may not resolve correctly at runtime. The intended behavior should use `={{ $json.company_name }}` as the left value.
- **Input:** Single job item from **Loop Over Job Batches**.
- **Output:**
  - Output 0 (true — company name IS empty): → **Detect Missing Company Names**
  - Output 1 (false — company name IS present): → **AI Job Scoring Agent**
- **Edge cases:**
  - ⚠️ **Bug alert:** The `leftValue` of `=[undefined]` likely does not evaluate `$json.company_name`. If this expression always resolves to a non-empty string (the literal text `[undefined]`), all jobs will route to output 1 (AI Scoring) and company enrichment will never trigger. This must be corrected to `={{ $json.company_name }}` for the intended logic to work.
  - If `company_name` contains whitespace only, the "empty" operator may not catch it; a trim would be needed.

#### Detect Missing Company Names
- **Type:** `@mendable/n8n-nodes-firecrawl.firecrawl` (v1)
- **Technical role:** Scrapes the job posting URL using Firecrawl to extract metadata (specifically `ogDescription`) as a fallback company name.
- **Configuration:**
  - Operation: `scrape`
  - URL: `={{ $json.job_url }}`
  - Scrape options: format = JSON, no custom headers
  - No authentication credential configured (Firecrawl may require an API key depending on the community node setup)
- **Key expressions/variables:** `$json.job_url`.
- **Input:** Job items where company name is missing (from **Check Company Name Presence** output 0).
- **Output:** Scraped page data including `data.metadata.ogDescription` → connects to **Append Jobs with Company Names**.
- **Edge cases:**
  - If `job_url` is invalid or the page is unreachable, Firecrawl returns an error or empty data.
  - `ogDescription` is often a page description, not a company name — it may contain job descriptions instead. The quality of this fallback varies significantly by job board.
  - Firecrawl API rate limits or timeouts.
  - Requires a Firecrawl API key to be configured in the node credentials (currently empty).

#### Append Jobs with Company Names
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Technical role:** Updates the Google Sheet row with the discovered company name (from Firecrawl's `ogDescription`), so that the AI scoring step has a company name to work with.
- **Configuration:**
  - Operation: `appendOrUpdate` (matches on `unique_id`)
  - Document ID: `1hex1u1cpI5sMx8I45ktgpZ4VhXReQA4EZdoCTVEc1bI`
  - Sheet Name (by ID): `1726393267`
  - Column mapping:
    - `unique_id` → `={{ $('Check for Company Name').item.json.unique_id }}`
    - `company_name` → `={{ $json.data.metadata.ogDescription }}`
  - Matching columns: `unique_id`
- **Key expressions/variables:** `$json.data.metadata.ogDescription`, `$('Check for Company Name').item.json.unique_id`.
- **Input:** Items from **Detect Missing Company Names**.
- **Output:** Same items → connects to **AI Job Scoring Agent**.
- **Credentials required:** Google Sheets OAuth2 API credential.
- **Edge cases:**
  - ⚠️ **Bug alert:** The expression references `$('Check for Company Name')` but the actual node name is **"Check Company Name Presence"**. This mismatch will cause a runtime error ("Node 'Check for Company Name' not found"). It must be corrected to `$('Check Company Name Presence')`.
  - If `ogDescription` is null or empty, the `company_name` column will be updated with an empty/null value.
  - If `ogDescription` contains HTML or lengthy text, it may not be a clean company name.

---

### Block 5 — AI Scoring & Lead Qualification

**Overview:** Each job is sent to OpenAI GPT-4o-mini with a detailed system prompt that acts as a lead qualification engine. The AI returns a structured JSON with `lead_score` (1–10), `decision` (ACCEPT/REJECT), `reason`, `industry`, and `country`. The response is parsed, and based on the decision, the job is either marked QUALIFIED (with a generated `company_id`) and saved, or marked NOT_QUALIFIED and saved. A NoOp node then cycles back to the job batch loop for the next job.

**Nodes Involved:** AI Job Scoring Agent, Parse AI Lead Scores, Check Lead Acceptance, Set Job as Qualified, Append Qualified Job to Sheets, Save Qualified Job to Sheets, Cycle Job Processing

#### AI Job Scoring Agent
- **Type:** `@n8n/n8n-nodes-langchain.openAi` (v2.1)
- **Technical role:** Invokes GPT-4o-mini as an LLM to score and classify each job lead.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Response format: JSON object (`textOptions.type: "json_object"`)
  - System prompt (detailed lead qualification engine):
    - **Context:** Manpower recruitment agency supplying hospitality, logistics, healthcare, oil & gas, and construction.
    - **Input fields injected:** Job Title, Job Location, Company Name, Platform, Job Description (via expressions `{{ $json.job_title }}`, `{{ $json.job_location }}`, `{{ $json.company_name }}`, `{{ $json.job_source }}`, `{{ $json.job_description }}`).
    - **Country extraction rules:** Maps location strings (including abbreviations like "DE", "QA", "UAE") to full country names.
    - **Scoring logic (1–10):**
      - 8–10 (High): Frontline/mid-level roles in target industries, GCC/Asian markets, active employer, recent post.
      - 5–7 (Medium): Senior roles, partially relevant industry, non-GCC valid markets.
      - 1–4 (Low): Unrelated roles (IT, Marketing, Finance), staffing agencies, fake/incomplete listings.
    - **Decision:** Score ≥ 7 → ACCEPT; Score < 7 → REJECT.
    - **Industry classification:** Must pick exactly one from a predefined list of 21 industries (Oil & Gas, Construction, Services, Logistics & Supply Chain, Manufacturing, Hospitality, Telecommunications & IT, Facility Management, Aviation / Airport, Retail, Shipbuilding, Automobile, Drivers & Operators, Furniture & Interiors, Garments & Furnishing, Agriculture, Community Services, Education, Healthcare, Other).
    - **Output format:** Strict JSON with keys `lead_score`, `decision`, `reason`, `industry`, `country`.
  - No built-in tools enabled.
- **Key expressions/variables:** `$json.job_title`, `$json.job_location`, `$json.company_name`, `$json.job_source`, `$json.job_description`.
- **Input:** Job items (either directly from **Check Company Name Presence** output 1 or via **Append Jobs with Company Names**).
- **Output:** LLM response object → connects to **Parse AI Lead Scores**.
- **Credentials required:** OpenAI API credential.
- **Edge cases:**
  - If the `json_object` response format is not respected, the AI may return markdown-wrapped JSON, causing parse failures downstream.
  - OpenAI rate limits or token limits — long job descriptions may exceed the model's context window.
  - If `job_description` is empty or null, the AI has less information and may default to low scores.
  - API key errors or quota exhaustion will halt the entire workflow.

#### Parse AI Lead Scores
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical role:** Extracts and normalizes the AI's JSON response, merges it with the original job data, and applies fallback values for any missing or malformed AI output.
- **Configuration:**
  - JavaScript code that:
    1. Reads the original job data from `$('Loop Over Job Batches').item.json`.
    2. Extracts the AI text from `raw.output[0].content[0].text`.
    3. If the text is an object, uses it directly; if a string, parses it with `JSON.parse`.
    4. On parse failure, falls back to `{ lead_score: 0, decision: 'REJECT', reason: 'Parse error', industry: 'Other', country: 'Unknown' }`.
    5. Merges all original job fields with the parsed AI fields.
    6. Ensures `company_name` comes from the original job data (not overwritten by AI).
    7. Adds a `company_source` debug field (`'job_data'` if company_name exists, `'missing'` otherwise).
- **Key expressions/variables:** `$('Loop Over Job Batches').item.json`, `raw.output[0].content[0].text`, `JSON.parse`.
- **Input:** Single AI response item from **AI Job Scoring Agent**.
- **Output:** Merged job+AI data object → connects to **Check Lead Acceptance**.
- **Edge cases:**
  - If the AI response structure changes (e.g., different content block format), the `output[0].content[0].text` path may fail, triggering the fallback.
  - The fallback assigns score 0 and REJECT, which is a safe default but means parse errors silently reject leads.
  - `company_name` is preserved from the job data even if the AI tried to modify it.

#### Check Lead Acceptance
- **Type:** `n8n-nodes-base.if` (v2.3)
- **Technical role:** Routes the job based on the AI's ACCEPT/REJECT decision.
- **Configuration:**
  - Combinator: AND
  - Condition: `$json.decision` equals "ACCEPT" (string comparison, case-sensitive, loose type validation).
- **Key expressions/variables:** `={{ $json.decision }}`.
- **Input:** Parsed lead data from **Parse AI Lead Scores**.
- **Output:**
  - Output 0 (true — decision == "ACCEPT"): → **Set Job as Qualified**.
  - Output 1 (false — decision != "ACCEPT"): → **Save Qualified Job to Sheets** (which records NOT_QUALIFIED).
- **Edge cases:**
  - Case sensitivity — if the AI returns "Accept" or "accept" instead of "ACCEPT", the condition will fail and the job will be rejected. The system prompt explicitly requests "ACCEPT"/"REJECT" but LLMs can deviate.
  - If `decision` is null/undefined, the loose type validation will evaluate to false, routing to the reject path.

#### Set Job as Qualified
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Technical role:** Enriches the job item with qualification metadata before writing to the sheet.
- **Configuration:**
  - `includeOtherFields: true` — preserves all existing fields from the input.
  - Assignments:
    - `status` → "QUALIFIED" (string)
    - `company_id` → `={{ ($json.company_name || '').toLowerCase().trim().replace(/\s+/g, '_') }}` — generates a slug from the company name as a simple identifier.
- **Key expressions/variables:** `$json.company_name` (for company_id generation).
- **Input:** ACCEPTED jobs from **Check Lead Acceptance** output 0.
- **Output:** Job items with added `status` and `company_id` → connects to **Append Qualified Job to Sheets**.
- **Edge cases:**
  - If `company_name` is empty, `company_id` will be an empty string.
  - The slug generation is simplistic — different casing of the same company name produces the same ID, but "Acme Corp" and "Acme Corporation" produce different IDs.

#### Append Qualified Job to Sheets
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Technical role:** Writes qualified lead data back to the main jobs sheet, updating the row (matched by `unique_id`) with qualification results.
- **Configuration:**
  - Operation: `appendOrUpdate`
  - Document ID: `1hex1u1cpI5sMx8I45ktgpZ4VhXReQA4EZdoCTVEc1bI`
  - Sheet Name (by ID): `1726393267`
  - Matching columns: `unique_id`
  - Column mapping:
    - `status` → `={{ $json.status }}` ("QUALIFIED")
    - `country` → `={{ $json.country }}`
    - `qualified` → `={{ $json.decision }}` ("ACCEPT")
    - `unique_id` → `={{ $json.unique_id }}`
    - `company_id` → `={{ $json.company_id }}`
    - `job_category` → `={{ $json.industry }}`
    - `qualification_reason` → `={{ $json.reason }}`
- **Key expressions/variables:** See mapping above.
- **Input:** Qualified jobs from **Set Job as Qualified**.
- **Output:** Same items → connects to **Cycle Job Processing**.
- **Credentials required:** Google Sheets OAuth2 API credential.
- **Edge cases:**
  - If `unique_id` doesn't match an existing row, a new row is appended (which could create a duplicate row rather than updating).
  - Google Sheets API rate limits.

#### Save Qualified Job to Sheets
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Technical role:** Records rejected (NOT_QUALIFIED) jobs in the same sheet, preserving the AI's scoring reasoning and classification for future analysis.
- **Configuration:**
  - Operation: `appendOrUpdate`
  - Document ID: `1hex1u1cpI5sMx8I45ktgpZ4VhXReQA4EZdoCTVEc1bI`
  - Sheet Name (by ID): `1726393267`
  - Matching columns: `unique_id`
  - Column mapping:
    - `status` → hardcoded "=NOT_QUALIFIED"
    - `country` → `={{ $json.country }}`
    - `qualified` → `={{ $json.decision }}` ("REJECT")
    - `unique_id` → `={{ $json.unique_id }}`
    - `job_category` → `={{ $json.industry }}`
    - `qualification_reason` → `={{ $json.reason }}`
- **Key expressions/variables:** See mapping above.
- **Input:** Rejected jobs from **Check Lead Acceptance** output 1.
- **Output:** Same items → connects to **Cycle Job Processing**.
- **Credentials required:** Google Sheets OAuth2 API credential.
- **Edge cases:**
  - Same `unique_id` matching concern as the qualified path.
  - No `company_id` is set for rejected jobs (the column is not mapped here).

#### Cycle Job Processing
- **Type:** `n8n-nodes-base.noOp` (v1)
- **Technical role:** A pass-through node that serves as a convergence point for both the qualified and rejected paths, routing all jobs back to the **Loop Over Job Batches** node for the next iteration.
- **Configuration:** No parameters.
- **Key expressions/variables:** None.
- **Input:** Items from **Append Qualified Job to Sheets** and **Save Qualified Job to Sheets**.
- **Output:** Same items → connects to **Loop Over Job Batches** (input index 0, which re-triggers the batch loop to emit the next job).
- **Edge cases:** None — purely a routing convenience node.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|----------------|----------------|-------------|
| When Job Form Submitted | formTrigger (v2.5) | Webhook entry point; renders search configuration form | — (trigger) | Parse Form Input Arrays | Input and parse form — Captures job search inputs and parses them for further processing. |
| Parse Form Input Arrays | code (v2) | Splits comma-separated roles/locations into cartesian product of search objects | When Job Form Submitted | Loop Over Search Batches | Input and parse form — Captures job search inputs and parses them for further processing. |
| Loop Over Search Batches | splitInBatches (v3) | Iterates through each role×location search combination | Parse Form Input Arrays; Cycle Job Processing (via Loop Over Job Batches done) | Submit Job Scrape Request (each batch); — (done, workflow ends) | Iterate through searches — Loops through parsed search terms and initiates the job scraping process. |
| Submit Job Scrape Request | httpRequest (v4.2) | Calls Apify actor synchronously to scrape job listings per search | Loop Over Search Batches | Flatten Job Results and ID | Scrape jobs and notify — Scrapes job listings using Apify and sends notification emails. |
| Flatten Job Results and ID | code (v2) | Normalizes Apify results, adds unique_id and job_hash for dedup | Submit Job Scrape Request | Read Jobs from Sheets | Flatten, deduplicate, and save — Processes scraped data, removes duplicates, and saves raw jobs to a sheet. |
| Read Jobs from Sheets | googleSheets (v4.7) | Reads all existing job rows from the dedup reference sheet | Flatten Job Results and ID | Identify Duplicate Jobs | Flatten, deduplicate, and save — Processes scraped data, removes duplicates, and saves raw jobs to a sheet. |
| Identify Duplicate Jobs | code (v2) | Filters out jobs whose hash already exists in the sheet | Read Jobs from Sheets | Append Raw Jobs to Sheets | Flatten, deduplicate, and save — Processes scraped data, removes duplicates, and saves raw jobs to a sheet. |
| Append Raw Jobs to Sheets | googleSheets (v4.7) | Appends new unique jobs to the main sheet with status "NEW" | Identify Duplicate Jobs | Loop Over Job Batches | Flatten, deduplicate, and save — Processes scraped data, removes duplicates, and saves raw jobs to a sheet. |
| Loop Over Job Batches | splitInBatches (v3) | Iterates through each saved job for individual processing | Append Raw Jobs to Sheets; Cycle Job Processing | Loop Over Search Batches (done); Check Company Name Presence (each item) | Loop each job and check company — Iterates through each saved job and checks for company name presence. |
| Check Company Name Presence | if (v2.3) | Routes jobs based on whether company_name is empty | Loop Over Job Batches | Detect Missing Company Names (empty); AI Job Scoring Agent (present) | Loop each job and check company — Iterates through each saved job and checks for company name presence. |
| Detect Missing Company Names | firecrawl (v1) | Scrapes job URL to extract ogDescription as fallback company name | Check Company Name Presence | Append Jobs with Company Names | Company name validation — Validates the presence of a company name and saves non-qualified jobs. |
| Append Jobs with Company Names | googleSheets (v4.7) | Updates sheet row with discovered company name from Firecrawl | Detect Missing Company Names | AI Job Scoring Agent | Company name validation — Validates the presence of a company name and saves non-qualified jobs. |
| AI Job Scoring Agent | openAi/langchain (v2.1) | GPT-4o-mini scores and classifies each job lead | Check Company Name Presence; Append Jobs with Company Names | Parse AI Lead Scores | AI score and lead qualification — Scores jobs using AI and determines lead qualification, saving qualified leads. |
| Parse AI Lead Scores | code (v2) | Parses AI JSON response, merges with job data, applies fallbacks | AI Job Scoring Agent | Check Lead Acceptance | AI score and lead qualification — Scores jobs using AI and determines lead qualification, saving qualified leads. |
| Check Lead Acceptance | if (v2.3) | Routes based on AI decision (ACCEPT vs REJECT) | Parse AI Lead Scores | Set Job as Qualified (ACCEPT); Save Qualified Job to Sheets (REJECT) | AI score and lead qualification — Scores jobs using AI and determines lead qualification, saving qualified leads. |
| Set Job as Qualified | set (v3.4) | Adds status=QUALIFIED and generates company_id slug | Check Lead Acceptance | Append Qualified Job to Sheets | AI score and lead qualification — Scores jobs using AI and determines lead qualification, saving qualified leads. |
| Append Qualified Job to Sheets | googleSheets (v4.7) | Updates sheet row for qualified leads with scoring results | Set Job as Qualified | Cycle Job Processing | AI score and lead qualification — Scores jobs using AI and determines lead qualification, saving qualified leads. |
| Save Qualified Job to Sheets | googleSheets (v4.7) | Updates sheet row for rejected leads with status NOT_QUALIFIED | Check Lead Acceptance | Cycle Job Processing | AI score and lead qualification — Scores jobs using AI and determines lead qualification, saving qualified leads. |
| Cycle Job Processing | noOp (v1) | Pass-through convergence node routing back to job batch loop | Append Qualified Job to Sheets; Save Qualified Job to Sheets | Loop Over Job Batches | AI score and lead qualification — Scores jobs using AI and determines lead qualification, saving qualified leads. |
| Sticky Note | stickyNote (v1) | Overview documentation | — | — | ### How it works: 1. Captures job search inputs via the form. 2. Parses inputs and initiates scraper for job listings. 3. Scrapes jobs and removes duplicates before saving. 4. Scores jobs using AI and determines lead qualification. 5. Saves qualified jobs and loops back for next batch. ### Setup steps: Ensure Apify API credentials are set for job scraping. Connect to Google Sheets API for saving and retrieving data. Configure OpenAI API credentials for AI scoring. Ensure Gmail API access for sending messages to be configured. ### Customization: Adjust the AI scoring logic in the 'AI Score and Qualify Lead' node to match specific business criteria. Make a copy of Gsheets: https://docs.google.com/spreadsheets/d/1-7nF5jwil_PenpLfLlqlTdDfBj5q4987drCZj2gkLrQ/edit?usp=sharing |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
1. **n8n instance** (self-hosted or cloud) with the following community nodes installed:
   - `@n8n/n8n-nodes-langchain` (for the OpenAI LLM node)
   - `@mendable/n8n-nodes-firecrawl` (for the Firecrawl scraping node)
2. **Apify account** with access to the `agentx/all-jobs-scraper` actor.
3. **Google Cloud project** with the Google Sheets API enabled and OAuth2 credentials configured.
4. **OpenAI account** with API access and a key for GPT-4o-mini.
5. **Firecrawl account** (if the community node requires an API key).
6. A copy of the reference Google Spreadsheet: https://docs.google.com/spreadsheets/d/1-7nF5jwil_PenpLfLlqlTdDfBj5q4987drCZj2gkLrQ/edit?usp=sharing

### Google Sheets Setup
7. Make a copy of the spreadsheet above into your own Google Drive.
8. Note the **Document ID** from the URL (the long string between `/d/` and `/edit`).
9. Identify the sheet tab used for reading existing jobs (dedup reference) — note its numeric **Sheet ID** (via the gid parameter in the URL or the Sheets API).
10. Identify the sheet tab used for writing/updating jobs — note its numeric **Sheet ID**.
11. Ensure your Google Sheets OAuth2 credential in n8n has edit access to this spreadsheet.

### Step-by-Step Node Creation

**Step 1 — Form Trigger**
- Add a **Form Trigger** node.
- Set Form Title: "Job Search Configuration".
- Set Form Description: "Configure job search parameters for lead generation".
- Add four form fields:
  - "Job Roles" — text, required, placeholder "Chef, Nurse, Driver, Welder"
  - "Locations" — text, required, placeholder "Kuwait, UAE, Germany"
  - "Max Results Per Search" — number, placeholder "Minimum 10"
  - "Posted Since" — dropdown with options: "24 hours", "3 days", "1 week", "2 weeks", "1 month"

**Step 2 — Parse Form Input Arrays**
- Add a **Code** node.
- Paste the JavaScript code that splits comma-separated roles and locations, builds the cartesian product, and returns an array of `{ job_role, location, max_results, posted_since }` objects.
- Connect: Form Trigger → Parse Form Input Arrays.

**Step 3 — Loop Over Search Batches**
- Add a **Split In Batches** node (v3).
- Set `reset` to `false`.
- Connect: Parse Form Input Arrays → Loop Over Search Batches (input 0).

**Step 4 — Submit Job Scrape Request**
- Add an **HTTP Request** node.
- Method: POST.
- URL: `https://api.apify.com/v2/acts/agentx~all-jobs-scraper/run-sync-get-dataset-items`
- Body specification: JSON.
- JSON Body (use expression mode):
  ```
  {
    "keyword": "{{ $json.job_role }}",
    "country": "{{ $json.location }}",
    "max_results": {{ $json.max_results }},
    "posted_since": "{{ $json.posted_since }}"
  }
  ```
- Timeout: 300000 ms.
- Authentication: Generic Credential Type → HTTP Header Auth.
- Create an `httpHeaderAuth` credential with your Apify API token (header name: `Authorization`, value: `Bearer <YOUR_APIFY_API_TOKEN>`).
- Connect: Loop Over Search Batches (output 1 — each batch) → Submit Job Scrape Request.

**Step 5 — Flatten Job Results and ID**
- Add a **Code** node.
- Paste the JavaScript code that flattens arrays, computes `simpleHash`, and adds `unique_id`, `job_hash`, `lead_source`, `search_role`, `search_location`.
- Connect: Submit Job Scrape Request → Flatten Job Results and ID.

**Step 6 — Read Jobs from Sheets**
- Add a **Google Sheets** node (v4.7).
- Operation: Read.
- Document: select your copied spreadsheet by ID.
- Sheet: select the dedup reference tab by ID.
- Output formatting: dates as FORMATTED_STRING, general as UNFORMATTED_VALUE.
- Enable "Always Output Data".
- Credential: select your Google Sheets OAuth2 credential.
- Connect: Flatten Job Results and ID → Read Jobs from Sheets.

**Step 7 — Identify Duplicate Jobs**
- Add a **Code** node.
- Paste the JavaScript code that builds a Set of existing `job_hash` values and filters out duplicates.
- Connect: Read Jobs from Sheets → Identify Duplicate Jobs.

**Step 8 — Append Raw Jobs to Sheets**
- Add a **Google Sheets** node (v4.7).
- Operation: Append (enable "Use Append").
- Document: same spreadsheet, by ID.
- Sheet: select the main jobs tab by ID.
- Mapping mode: "Define Below".
- Map columns:
  - `status` → "NEW"
  - `job_url` → `{{ $json.platform_url }}`
  - `job_type` → `{{ $json.job_type }}`
  - `job_title` → `{{ $json.title }}`
  - `unique_id` → `{{ $json.unique_id }}`
  - `created_at` → `{{ $now }}`
  - `job_source` → `{{ $json.platform }}`
  - `company_name` → `{{ $json.company_name }}`
  - `job_location` → `{{ $json.location }}`
  - `original_json` → `{{ JSON.stringify($json) }}`
  - `job_description` → `{{ $json.description }}`
  - `job_posted_date` → `{{ $json.posted_date }}`
- Define the full schema (all 16 columns including company_id, job_category, qualified, qualification_reason) to ensure the sheet headers are created correctly.
- Connect: Identify Duplicate Jobs → Append Raw Jobs to Sheets.

**Step 9 — Loop Over Job Batches**
- Add a **Split In Batches** node (v3).
- Set `reset` to `false`.
- Connect: Append Raw Jobs to Sheets → Loop Over Job Batches (input 0).
- Also connect: Loop Over Job Batches output 0 (done) → Loop Over Search Batches (input 0). This creates the outer loop cycle.

**Step 10 — Check Company Name Presence**
- Add an **IF** node (v2.3).
- Combinator: AND.
- Condition: operator `empty` on `leftValue`. ⚠️ **Correct the expression**: set leftValue to `={{ $json.company_name }}` (not `=[undefined]`).
- Enable loose type validation.
- Connect: Loop Over Job Batches (output 1 — each item) → Check Company Name Presence.
- True output (company name empty) → Detect Missing Company Names.
- False output (company name present) → AI Job Scoring Agent.

**Step 11 — Detect Missing Company Names**
- Add a **Firecrawl** node (`@mendable/n8n-nodes-firecrawl.firecrawl`, v1).
- Operation: Scrape.
- URL: `={{ $json.job_url }}`
- Format: JSON.
- Configure Firecrawl API credentials if required.
- Connect: Check Company Name Presence (output 0) → Detect Missing Company Names.

**Step 12 — Append Jobs with Company Names**
- Add a **Google Sheets** node (v4.7).
- Operation: Append or Update.
- Document: same spreadsheet.
- Sheet: main jobs tab.
- Matching columns: `unique_id`.
- Map columns:
  - `unique_id` → `={{ $('Check Company Name Presence').item.json.unique_id }}` ⚠️ **Use the exact node name** (not "Check for Company Name").
  - `company_name` → `={{ $json.data.metadata.ogDescription }}`
- Connect: Detect Missing Company Names → Append Jobs with Company Names.
- Connect: Append Jobs with Company Names → AI Job Scoring Agent.

**Step 13 — AI Job Scoring Agent**
- Add an **OpenAI** node (`@n8n/n8n-nodes-langchain.openAi`, v2.1).
- Model: `gpt-4o-mini`.
- Response format: JSON object.
- System prompt: Paste the full lead qualification engine prompt (see Block 5 analysis above for the complete text), ensuring the `{{ $json.* }}` expressions for job_title, job_location, company_name, job_source, and job_description are included.
- Credential: select or create your OpenAI API credential.
- Connect: Check Company Name Presence (output 1) → AI Job Scoring Agent.
- Also connect: Append Jobs with Company Names → AI Job Scoring Agent.

**Step 14 — Parse AI Lead Scores**
- Add a **Code** node.
- Paste the JavaScript code that reads from `$('Loop Over Job Batches').item.json`, parses `raw.output[0].content[0].text`, and merges with fallback defaults.
- Connect: AI Job Scoring Agent → Parse AI Lead Scores.

**Step 15 — Check Lead Acceptance**
- Add an **IF** node (v2.3).
- Condition: `={{ $json.decision }}` equals "ACCEPT" (string, case-sensitive).
- Enable loose type validation.
- Connect: Parse AI Lead Scores → Check Lead Acceptance.
- True (ACCEPT) → Set Job as Qualified.
- False (REJECT or other) → Save Qualified Job to Sheets.

**Step 16 — Set Job as Qualified**
- Add a **Set** node (v3.4).
- Enable "Include Other Fields".
- Assignments:
  - `status` → "QUALIFIED"
  - `company_id` → `={{ ($json.company_name || '').toLowerCase().trim().replace(/\s+/g, '_') }}`
- Connect: Check Lead Acceptance (output 0) → Set Job as Qualified.

**Step 17 — Append Qualified Job to Sheets**
- Add a **Google Sheets** node (v4.7).
- Operation: Append or Update.
- Document: same spreadsheet.
- Sheet: main jobs tab.
- Matching columns: `unique_id`.
- Map columns: `status`, `country`, `qualified`, `unique_id`, `company_id`, `job_category`, `qualification_reason` (see Block 5 for expressions).
- Connect: Set Job as Qualified → Append Qualified Job to Sheets.

**Step 18 — Save Qualified Job to Sheets (rejected path)**
- Add a **Google Sheets** node (v4.7).
- Operation: Append or Update.
- Document: same spreadsheet.
- Sheet: main jobs tab.
- Matching columns: `unique_id`.
- Map columns: `status` → "=NOT_QUALIFIED", `country`, `qualified`, `unique_id`, `job_category`, `qualification_reason` (see Block 5 for expressions).
- Connect: Check Lead Acceptance (output 1) → Save Qualified Job to Sheets.

**Step 19 — Cycle Job Processing**
- Add a **NoOp** node.
- Connect: Append Qualified Job to Sheets → Cycle Job Processing.
- Connect: Save Qualified Job to Sheets → Cycle Job Processing.
- Connect: Cycle Job Processing → Loop Over Job Batches (input 0).

**Step 20 — Verify all connections**
Confirm the following flow paths:
- Form → Parse → Search Loop → Scrape → Flatten → Read Sheet → Dedupe → Append Raw → Job Loop → Company Check → [Firecrawl → Update Sheet →] AI Score → Parse → Accept/Reject → [Set Qualified →] Sheet Update → Cycle → Job Loop (next) → … → Search Loop (next) → … → End.

### Credential Summary

| Credential Type | Purpose | Required For |
|-----------------|---------|--------------|
| HTTP Header Auth (Apify) | Apify API token for the job scraper actor | Submit Job Scrape Request |
| Google Sheets OAuth2 API | Read/write access to the spreadsheet | Read Jobs, Append Raw Jobs, Append Jobs with Company Names, Append Qualified Job, Save Qualified Job |
| OpenAI API | GPT-4o-mini model access | AI Job Scoring Agent |
| Firecrawl API (if required) | Web scraping for company name enrichment | Detect Missing Company Names |

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Make a copy of the Google Sheets template before running the workflow; update all Document ID and Sheet ID references to point to your own copy. | https://docs.google.com/spreadsheets/d/1-7nF5jwil_PenpLfLlqlTdDfBj5q4987drCZj2gkLrQ/edit?usp=sharing |
| The Apify actor used is `agentx/all-jobs-scraper` — ensure your Apify plan includes sufficient compute units for the expected search volume. | https://apify.com/agentx/all-jobs-scraper |
| The Firecrawl `ogDescription` fallback for company names is unreliable — many job boards use `og:description` for the job description rather than the company name. Consider adding a dedicated company name extraction prompt to the AI scoring step instead. | General note |
| The **Check Company Name Presence** IF node has a misconfigured `leftValue` of `=[undefined]`. This must be corrected to `={{ $json.company_name }}` for the company enrichment branch to trigger correctly. | Bug fix |
| The **Append Jobs with Company Names** node references `$('Check for Company Name')` which does not match any node name. Correct it to `$('Check Company Name Presence')`. | Bug fix |
| The `simpleHash` function in the Flatten node is not cryptographically collision-resistant. For production use with very large datasets, consider replacing it with a SHA-256 hash via the `crypto` module. | General note |
| The sticky note mentions "Gmail API access for sending messages to be configured," but no Gmail node exists in the current workflow. This appears to be a planned feature not yet implemented. | Setup checklist note |
| The AI scoring prompt enforces a score ≥ 7 threshold for ACCEPT. Adjust the threshold in the system prompt and/or the Check Lead Acceptance condition to match different business criteria. | Customization |
| The synchronous Apify call (`run-sync-get-dataset-items`) blocks the workflow for up to 5 minutes per search. For large search volumes, consider switching to the asynchronous Apify API (run → poll for completion → fetch dataset) to avoid n8n execution timeouts. | Performance note |
| All Google Sheets writes in this workflow operate on the same document ID and sheet ID (`1726393267`). The Read node uses a different sheet ID (`1451193517`) in the same document, likely a dedicated "All Jobs" or dedup tab. Ensure both tabs exist in your spreadsheet copy. | Sheet structure note |