Aggregate multi-source job boards to Supabase and Google Sheets

https://n8nworkflows.xyz/workflows/aggregate-multi-source-job-boards-to-supabase-and-google-sheets-14996


# Aggregate multi-source job boards to Supabase and Google Sheets

I need to analyze the workflow JSON thoroughly and produce a comprehensive documentation. Let me parse the structure carefully.

**Nodes:**

1. **Main Sticky** - stickyNote, color 2, large note describing the workflow
2. **Company List** - code node, defines sources array and returns items
3. **Prepare Request** - code node, builds URLs and headers for each source, including pagination
4. **HTTP Request** - httpRequest node, fetches data from URLs, batches of 5
5. **Upsert to Supabase** - postgres node, upserts to table
6. **Write to Google Sheet** - googleSheets node, appendOrUpdate
7. **Loop Batches (5)** - splitInBatches node, batchSize 5
8. **Daily 8AM IST** - scheduleTrigger, 8AM
9. **Parse and Enrich and Filter** - code node, large parsing logic
10. **Trigger and Config** - stickyNote, section 1
11. **Parsing and Enrichment** - stickyNote, section 2
12. **Destination** - stickyNote, section 3

**Connections:**

- Daily 8AM IST → Company List
- Company List → Loop Batches (5)
- Loop Batches (5) → Prepare Request (second output)
- Prepare Request → HTTP Request
- HTTP Request → Parse and Enrich and Filter
- Parse and Enrich and Filter → Upsert to Supabase
- Upsert to Supabase → Write to Google Sheet
- Write to Google Sheet → Loop Batches (5) (loops back)

So the flow is:
1. Schedule Trigger fires at 8AM IST
2. Company List code node creates source items
3. Loop Batches (5) splits items into batches of 5
4. For each batch: Prepare Request builds URLs → HTTP Request fetches → Parse and Enrich and Filter normalizes
5. After parsing, Upsert to Supabase → Write to Google Sheet
6. After writing, loops back to process next batch

Wait, actually let me look at the connections more carefully:

```
"Company List":{"main":[[{"node":"Loop Batches (5)","type":"main","index":0}]]}
"HTTP Request":{"main":[[{"node":"Parse and Enrich and Filter","type":"main","index":0}]]}
"Daily 8AM IST":{"main":[[{"node":"Company List","type":"main","index":0}]]}
"Prepare Request":{"main":[[{"node":"HTTP Request","type":"main","index":0}]]}
"Loop Batches (5)":{"main":[[],[{"node":"Prepare Request","type":"main","index":0}]]}
"Upsert to Supabase":{"main":[[{"node":"Write to Google Sheet","type":"main","index":0}]]}
"Write to Google Sheet":{"main":[[{"node":"Loop Batches (5)","type":"main","index":0}]]}
"Parse and Enrich and Filter":{"main":[[{"node":"Upsert to Supabase","type":"main","index":0}]]}
```

So Loop Batches has two outputs: output 0 (done) and output 1 (each batch). Output 1 goes to Prepare Request. After Write to Google Sheet, it loops back to Loop Batches (5).

The SplitInBatches node works like:
- First input receives items
- Output 1 emits batches of 5
- After processing, items go back to the node, and when all batches processed, output 0 emits "done"

So the flow per batch:
1. Prepare Request builds URLs for the 5 sources in this batch
2. HTTP Request fetches all 5
3. Parse and Enrich and Filter processes all results
4. Upsert to Supabase
5. Write to Google Sheet
6. Loop back to process next batch

Now let me think about the logical blocks:

**Block 1: Trigger and Source Configuration**
- Daily 8AM IST (Schedule Trigger)
- Company List (Code node defining sources)

**Block 2: Batch Execution and Request Preparation**
- Loop Batches (5) (SplitInBatches)
- Prepare Request (Code node building URLs)

**Block 3: Data Fetching and Parsing**
- HTTP Request (fetching data)
- Parse and Enrich and Filter (massive code node)

**Block 4: Data Storage**
- Upsert to Supabase (Postgres node)
- Write to Google Sheet (Google Sheets node)

Now let me document each node in detail.

---

**Sticky Notes:**

1. Main Sticky - Large note with workflow description, how it works, setup, customization
2. Trigger and Config - "## 1. Source Discovery and Trigger\nDefines target ATS sources and schedules the daily sync process."
3. Parsing and Enrichment - "## 2. Request and Normalization\nExecutes HTTP requests and uses advanced logic to map inconsistent ATS payloads to a standard schema."
4. Destination - "## 3. Storage and Output\nUpserts standardized job data into Supabase and updates the target Google Sheet."

These sticky notes visually cover nodes:
- Trigger and Config covers: Daily 8AM IST, Company List
- Parsing and Enrichment covers: Loop Batches (5), Prepare Request, HTTP Request, Parse and Enrich and Filter
- Destination covers: Upsert to Supabase, Write to Google Sheet
- Main Sticky is the overall description

Let me now write the full documentation.

For the summary table, I need to map each node to its sticky note(s). Based on positions:

- Main Sticky at [-2368, 2640] covers the whole workflow area
- Trigger and Config at [-1824, 2640] with width 668, height 280 — covers from x=-1824 to -1156, y=2640 to 2920. This covers Daily 8AM IST (at [-1792, 2736]) and Company List (at [-1568, 2736]).
- Parsing and Enrichment at [-1104, 2640] with width 684, height 280 — covers from x=-1104 to -420, y=2640 to 2920. This covers Loop Batches (5) at [-1344, 2736] — wait, that's outside this range (-1344 < -1104). Hmm, let me check more carefully.

Actually the positions:
- Loop Batches (5): [-1344, 2736]
- Prepare Request: [-1056, 2752]
- HTTP Request: [-832, 2752]
- Parse and Enrich and Filter: [-592, 2752]

The Parsing and Enrichment note spans from x=-1104 to x=-420, which covers Prepare Request, HTTP Request, and Parse and Enrich and Filter. Loop Batches is at x=-1344 which is outside that range.

Actually, looking at the layout more carefully:
- Trigger and Config: [-1824, 2640], w=668, h=280 → x range: -1824 to -1156
- Parsing and Enrichment: [-1104, 2640], w=684, h=280 → x range: -1104 to -420
- Destination: [-384, 2640], w=600, h=280 → x range: -384 to 216

Node positions:
- Daily 8AM IST: [-1792, 2736] → in Trigger and Config zone
- Company List: [-1568, 2736] → in Trigger and Config zone
- Loop Batches (5): [-1344, 2736] → hmm, -1344 is between -1824 and -1156? No, -1344 is outside Trigger and Config (which goes to -1156). But -1344 is also outside Parsing and Enrichment (which starts at -1104). So Loop Batches sits between the two zones.

Actually let me reconsider. The x-coordinate -1344 is less than -1104, so it's not in the Parsing and Enrichment zone. And -1344 is greater than -1156? No, -1344 < -1156 (more negative). So -1344 is to the left of -1156, meaning it's still within the Trigger and Config zone? Let me check: Trigger and Config starts at x=-1824, ends at x=-1824+668=-1156. Loop Batches is at x=-1344, which is between -1824 and -1156. Yes! So Loop Batches is within Trigger and Config.

OK so:
- Trigger and Config covers: Daily 8AM IST, Company List, Loop Batches (5)
- Parsing and Enrichment covers: Prepare Request, HTTP Request, Parse and Enrich and Filter
- Destination covers: Upsert to Supabase, Write to Google Sheet
- Main Sticky is an overall note

Let me verify the Parsing and Enrichment: starts at -1104, width 684, so ends at -420. Prepare Request at -1056 ✓, HTTP Request at -832 ✓, Parse and Enrich at -592 ✓.

Destination: starts at -384, width 600, ends at 216. Upsert to Supabase at -272 ✓, Write to Google Sheet at -16 ✓.

Great, now let me map each node to its sticky notes.

Now let me write the comprehensive documentation.

Let me think about each node in detail:

### Daily 8AM IST
- Type: Schedule Trigger
- Triggers at 8 AM (server timezone? It says IST but n8n schedule triggers use server timezone)
- Output: single empty item to trigger the workflow

### Company List
- Type: Code (JavaScript)
- Defines a `sources` array with 42 sources
- Returns each source as an item with json: {ats, slug, company}
- Sources include: remotive, himalayas, arbeitnow, jobicy (2 variants), greenhouse (21 companies), smartrecruiters (3), ashby (8), workable (3), lever (6), remoteok (1)
- Version 2 of the code node

### Loop Batches (5)
- Type: SplitInBatches
- Batch size: 5
- Takes input items and processes them 5 at a time
- Output 0: done (when all batches processed)
- Output 1: current batch items

### Prepare Request
- Type: Code (JavaScript)
- Takes batch items and builds URL + headers for each
- Handles pagination for SmartRecruiters (5 pages × 100 = 500) and Himalayas (10 pages × 50 = 500)
- Returns items with _url, _headers, _route, and original source metadata
- Headers include User-Agent and Accept

### HTTP Request
- Type: HTTP Request
- URL: expression from $json._url
- Headers: expression from $json._headers (JSON stringified)
- Batching: batchSize 5
- Timeout: 30000ms
- Error handling: continue on error

### Parse and Enrich and Filter
- Type: Code (JavaScript)
- Massive parsing logic
- Detects ATS type from response structure
- Normalizes each ATS's JSON into unified schema
- Applies filtering: ALLOWED_DOMAINS for job titles, keeps India-based or Remote jobs
- Deduplicates by job_hash
- Outputs: job_title, company, location, country, work_mode, employment_type, apply_url, ats_type, job_hash, description, salary, posted_date, status

### Upsert to Supabase
- Type: Postgres
- Operation: upsert
- Table: YOUR_TABLE_NAME (placeholder)
- Schema: public
- Matching column: job_hash
- Maps fields: salary, status, company, country, ats_type, job_hash, location, apply_url, job_title, work_mode, description, employment_type
- Error handling: continue on error

### Write to Google Sheet
- Type: Google Sheets
- Operation: appendOrUpdate
- Matching column: job_hash
- Document ID: YOUR_RESOURCE_ID_HERE (placeholder)
- Sheet name: YOUR_RESOURCE_ID_HERE (placeholder)
- Columns: job_hash, Job Title, Company, Location, Country, Work Mode, Employment Type, Apply URL, ATS, Salary, Status, Description
- References Parse and Enrich and Filter node for values

Now let me compose the full documentation.

I need to be very thorough about the Parse node since it's the most complex. Let me also think about edge cases and potential failures.

Edge cases for Parse node:
- Lever and RemoteOK return bare arrays, which n8n splits into individual items - the parser handles this with pairedItem to match back to config
- SmartRecruiters and Himalayas pagination creates multiple items per source
- HTML/JSON-LD parsing with regex may miss some entries
- Salary parsing handles many formats
- Country detection has India city detection, ISO code mapping
- Filtering keeps only India or Remote jobs
- Domain filtering checks job titles against ALLOWED_DOMAINS list

Potential failures:
- HTTP timeout (30s) - continues on error
- Invalid JSON responses - some ATS may change API structure
- Rate limiting from ATS APIs
- Database connection failures
- Google Sheets API rate limits
- Missing or null fields in responses

Let me also note that the "Write to Google Sheet" node references `$('Parse and Enrich and Filter').item.json` for each column, which means it reads from the Parse node's output for each item being processed. Since this node runs after Upsert to Supabase, and both receive the same items from Parse, this should work.

Now let me write the full documentation following the required structure.

I'll organize into logical blocks:

**Block 1: Source Discovery and Trigger**
- Daily 8AM IST
- Company List
- Loop Batches (5)

**Block 2: Request Building and Data Fetching**
- Prepare Request
- HTTP Request

**Block 3: Parsing, Enrichment, and Filtering**
- Parse and Enrich and Filter

**Block 4: Storage and Output**
- Upsert to Supabase
- Write to Google Sheet

Now I'll write it all out.

Let me also think about the connections carefully:

1. Daily 8AM IST → Company List (trigger fires, produces empty item, Company List runs)
2. Company List → Loop Batches (5) (all source items go in)
3. Loop Batches (5) → Prepare Request (batch of 5 items goes to Prepare Request via output 1)
4. Prepare Request → HTTP Request (URL items go to HTTP Request)
5. HTTP Request → Parse and Enrich and Filter (raw responses go to parser)
6. Parse and Enrich and Filter → Upsert to Supabase (normalized jobs)
7. Upsert to Supabase → Write to Google Sheet
8. Write to Google Sheet → Loop Batches (5) (loops back for next batch)

Wait, I need to think about this more carefully. The SplitInBatches node receives items from Company List. It then outputs batches of 5 to Prepare Request. Prepare Request expands pagination items. HTTP Request fetches each URL. Parse processes all results from that batch. Then Upsert and Write happen. Then it loops back.

But there's a subtlety: the Prepare Request node takes 5 source items and may produce more than 5 items (due to pagination for SmartRecruiters and Himalayas). So the HTTP Request node will fetch all of those URLs. Then Parse processes all results.

The SplitInBatches ensures we process 5 source configurations at a time, but each source may have multiple pages. This is to avoid overwhelming the APIs.

Also, the HTTP Request has batching enabled with batchSize 5, which means it processes up to 5 HTTP requests concurrently per batch. This is at the node level, separate from the SplitInBatches.

Now for the summary table, I need to list all nodes with their sticky notes.

Let me write the documentation now.

One more consideration: the Parse node's `pairedItem` matching. Since n8n splits bare arrays from Lever and RemoteOK, the HTTP Request may produce multiple items for one source. The Parse node uses `pairedItem` to match back to the configuration. This is a key complexity point.

Also, the Parse node reads configuration from `$('Prepare Request').all()` which contains all the prepared request items (with _url, _headers, etc.) for the current batch. It then matches each HTTP response to its corresponding configuration using `pairedItem`.

Let me make sure I document this correctly.

Now I'll compose the complete output.### 1. Workflow Overview

This workflow automatically aggregates job postings from **8+ public ATS platforms and job boards**, normalises every listing into a single clean schema, and syncs the deduplicated results to **Supabase (Postgres)** and **Google Sheets**. It is designed to run unattended every morning, requiring no API keys or web scraping.

**Logical blocks**

| # | Block | Purpose | Nodes |
|---|---|---|---|
| 1 | Source Discovery & Trigger | Starts the workflow daily and defines which ATS sources/companies to query | Daily 8AM IST, Company List, Loop Batches (5) |
| 2 | Request Building & Data Fetching | Constructs the correct API URL, headers, and pagination for each source, then fetches the data | Prepare Request, HTTP Request |
| 3 | Parsing, Enrichment & Filtering | Detects the ATS type from the response, normalises divergent JSON shapes into a unified schema, resolves countries/salaries/work modes, applies domain and geography filters, and deduplicates | Parse and Enrich and Filter |
| 4 | Storage & Output | Upserts normalised jobs into Supabase (matching on `job_hash`) and appends/updates them in a Google Sheet | Upsert to Supabase, Write to Google Sheet |

The workflow loops: the SplitInBatches node processes the full source list **5 sources at a time**, which limits concurrent HTTP calls and keeps memory usage bounded.

---

### 2. Block-by-Block Analysis

---

#### Block 1 – Source Discovery & Trigger

**Overview**  
A cron-style trigger fires the workflow once per day. A code node then emits one item per ATS source (company + platform), and a batch splitter controls the throughput by feeding only 5 sources into the request pipeline at a time.

**Nodes Involved**  
- Daily 8AM IST  
- Company List  
- Loop Batches (5)

---

**Daily 8AM IST**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.scheduleTrigger` |
| Version | 1.1 |
| Configuration | Fires once per day at **08:00** (server timezone; described as IST in the workflow title) |
| Output | A single empty item that kicks off the chain |
| Edge cases | If the n8n server timezone differs from IST, the actual fire time will shift accordingly. DST changes may also affect timing. |

---

**Company List**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.code` (JavaScript) |
| Version | 2 |
| Functional role | Declares the **sources** array of 42 entries, each with `ats`, `slug`, and `company`. Returns one item per source. |
| Configuration | The `sources` array is hard-coded inside `jsCode`. Grouped as: global board APIs (Remotive, Himalayas, Arbeitnow, Jobicy × 2), Greenhouse (21 slugs), SmartRecruiters (3), Ashby (8), Workable (3), Lever (6), RemoteOK (1). |
| Key expressions | None – pure JavaScript. |
| Input | Single trigger item from Schedule node. |
| Output | 42 items, each shaped `{ ats: 'greenhouse', slug: 'gitlab', company: 'GitLab' }` etc. |
| Edge cases / failures | • If a slug is wrong the downstream HTTP call will 404; the HTTP node is set to **continue on error** so the workflow won't stop. <br>• Adding new ATS types requires edits here *and* in the Parse node. |
| Sub-workflow | None |

---

**Loop Batches (5)**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.splitInBatches` |
| Version | 3 |
| Functional role | Splits the incoming source list into **batches of 5**. Output 1 emits the current batch; after downstream processing finishes, control loops back here for the next batch. Output 0 fires once all batches are done (currently unused). |
| Configuration | `batchSize: 5` |
| Input | 42 items from Company List. |
| Output 1 → | Prepare Request (current batch) |
| Output 0 (done) | Not connected – workflow ends naturally after the last batch loop. |
| Edge cases | If fewer than 5 sources remain, a smaller batch is emitted. The node resets automatically between workflow executions. |

---

#### Block 2 – Request Building & Data Fetching

**Overview**  
For every source in the current batch, a code node constructs the exact API endpoint and required HTTP headers—including multi-page URLs for SmartRecruiters and Himalayas. The HTTP Request node then calls all endpoints, processing up to 5 in parallel per node-level batching.

**Nodes Involved**  
- Prepare Request  
- HTTP Request

---

**Prepare Request**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.code` (JavaScript) |
| Version | 2 |
| Functional role | Transforms each source configuration into a full request descriptor: `_url`, `_headers`, `_route`, plus the original `ats`, `slug`, `company`. Handles pagination by emitting multiple items for paginated sources. |
| Configuration | • Default headers: `Accept: application/json`, `User-Agent: Mozilla/5.0 (compatible; JobBot/4.2)` <br>• RemoteOK gets an additional `Cache-Control: no-cache` header <br>• `html_json_ld` sources receive a browser-like User-Agent and `Accept: text/html` |
| Key expressions | None in the node itself; the next node reads `$json._url` and `$json._headers`. |
| Pagination logic | • **SmartRecruiters**: 5 pages × 100 offset = 500 jobs max. Emits 5 items per company slug. <br>• **Himalayas**: 10 pages × 50 = 500 jobs max. Emits 10 items per slug. <br>• All other sources: single URL per slug. |
| URL map (non-paginated) | remotive → `https://remotive.com/api/remote-jobs?limit=500` <br>remoteok → `https://remoteok.com/api` <br>arbeitnow → `https://www.arbeitnow.com/api/job-board-api` <br>jobicy (India) → `https://jobicy.com/api/v2/remote-jobs?count=100&geo=india` <br>jobicy (USA) → `https://jobicy.com/api/v2/remote-jobs?count=100&geo=usa` <br>lever → `https://api.lever.co/v0/postings/{slug}?mode=json&limit=250` <br>greenhouse → `https://boards-api.greenhouse.io/v1/boards/{slug}/jobs?content=true` <br>workable → `https://apply.workable.com/api/v1/widget/accounts/{slug}` <br>ashby → `https://api.ashbyhq.com/posting-api/job-board/{slug}` |
| Input | Up to 5 items from Loop Batches (5). |
| Output | Potentially >5 items due to pagination expansion (e.g., a batch with 1 SmartRecruiters + 4 others → 5 + 4 × 1 = 9 items). Each item: `{ ats, slug, company, _url, _headers, _route }`. |
| Edge cases | • Unknown `ats` values are logged and skipped. <br>• If a custom URL is provided for `html_json_ld`, it is passed through unchanged. <br>• Adding a new ATS requires a URL map entry here and parser logic in the Parse node. |

---

**HTTP Request**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.httpRequest` |
| Version | 4.2 |
| Functional role | Calls every `_url` from the previous node, returning raw JSON (or HTML) responses. |
| Configuration | • URL: `={{ $json._url }}` <br>• Headers: `={{ JSON.stringify($json._headers) }}` (sent as a JSON header block) <br>• `sendHeaders: true`, `specifyHeaders: json` <br>• Timeout: 30 000 ms <br>• Batching: `batchSize: 5` (node-level parallelism) <br>• Error handling: `continueRegularOutput` – the workflow proceeds even if one request fails |
| Input | Items from Prepare Request (each carrying `_url` and `_headers`). |
| Output | One item per URL, containing the full response body. For Lever and RemoteOK the response is a **bare JSON array**, which n8n auto-splits into individual items; this is handled downstream. |
| Edge cases | • Timeouts after 30 s – the item will be empty/error but execution continues. <br>• Rate limiting / 429 from ATS – same graceful handling. <br>• Redirects are followed by default. <br>• Some endpoints return HTML (JSON-LD pages); these arrive as strings. |

---

#### Block 3 – Parsing, Enrichment & Filtering

**Overview**  
A single large code node inspects every HTTP response, auto-detects which ATS produced it, normalises the divergent payload shapes into one schema, enriches data (country detection, salary formatting, work-mode inference), applies allow-list and geography filters, and deduplicates entries by a deterministic `job_hash`.

**Nodes Involved**  
- Parse and Enrich and Filter

---

**Parse and Enrich and Filter**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.code` (JavaScript) |
| Version | 2 |
| Functional role | Core transformation engine. Detects ATS type, extracts job fields, normalises them, applies business rules, and emits a flat list of standardised job items. |
| Key sub-functions | **Helper utilities** – `str()`, `parseSalary()`, `fmt()`, `parseDate()`, `getCountry()`, `getWorkMode()`, `cleanLoc()`, `normType()`, `cleanDesc()`, `makeHash()`. <br>**ATS detection** – `detectATS(resp)` inspects the shape of the response object/array to determine the platform. <br>**Company inference** – `getCompanyFromResp()` extracts the real company name from Lever hostedUrl, Greenhouse absolute_url, Ashby jobUrl, or falls back to the configured `company`/`slug`. <br>**Per-ATS parsers** – dedicated `switch` cases for `remotive`, `remoteok`, `himalayas`, `arbeitnow`, `jobicy`, `lever`, `greenhouse`, `workable`, `smartrecruiters`, `ashby`, and `html_json_ld`. <br>**Filtering** – `ALLOWED_DOMAINS` list of title keywords (product manager, data analyst, UX designer, full stack, sales, etc.). `isTitleAllowed()` checks inclusion. `shouldKeep()` retains only jobs where country is India **or** work_mode is Remote. <br>**Deduplication** – `makeHash(ats, company, title, location)` produces a deterministic hash; a `Set` prevents duplicates within a single run. |
| Output schema (per item) | `job_title`, `company`, `location`, `country`, `work_mode`, `employment_type`, `apply_url`, `ats_type`, `job_hash`, `description` (HTML-stripped, max 2000 chars), `salary` (formatted string like `$80k--$120k/yr USD`), `posted_date` (ISO date), `status` (always `'active'`). |
| Salary parsing | Handles strings (`"$80,000 - $120,000"`, `"INR 10L - 20L"`), numbers, and objects with `min`/`max`/`currency`/`interval`. Indian numbers are formatted with L (lakh) and Cr (crore) suffixes. |
| Country detection | First checks raw ISO2 codes, then full country names, then Indian city names from a 31-entry set, then pattern matching against a `CNAMES` dictionary, and finally a fallback to `'Unknown'`. The special value `'Global'` is assigned for worldwide/remote listings. |
| Work mode inference | Checks explicit fields first, then scans title + location for keywords (`remote`, `hybrid`, `wfh`, `distributed`). Defaults to `'Onsite'`. |
| Employment type normalisation | Maps `"intern"`, `"contract"`, `"part-time"` to `Internship`, `Contract`, `Part-time`; everything else becomes `Full-time`. |
| Location cleaning | Standardises city names (Bangalore → Bengaluru, Bombay → Mumbai, etc.), trims whitespace, replaces multiple commas. |
| HTML / JSON-LD parsing | For `html_json_ld` responses: regex extracts `<script type="application/ld+json">` blocks, parses them, and walks `@graph` arrays to find `JobPosting` entries. |
| Paired-item matching | The node reads `$input.all()` (HTTP responses) and `$('Prepare Request').all()` (config items). It uses `pairedItem` from the HTTP response to find the correct configuration entry, falling back to index-based matching. This is critical when Lever/RemoteOK responses are split by n8n into many items. |
| Input | All HTTP response items from the current batch. |
| Output | An array of normalised job objects; if no jobs pass the filter, a single `{ _empty: true }` item is returned to prevent downstream errors. |
| Edge cases / failures | • If `detectATS()` returns `null`, the item is skipped with a console log. <br>• Malformed JSON-LD is caught in a try/catch. <br>• Bare arrays from Lever/RemoteOK are handled via the split-detection logic. <br>• The `_empty: true` sentinel ensures the Postgres and Sheets nodes receive at least one item, avoiding "no data" errors. <br>• Future ATS additions require a new `case` block and a URL entry in Prepare Request. |

---

#### Block 4 – Storage & Output

**Overview**  
Normalised and filtered jobs are upserted into a Supabase Postgres table keyed on `job_hash`, and simultaneously appended (or updated) into a Google Sheet that mirrors the same schema.

**Nodes Involved**  
- Upsert to Supabase  
- Write to Google Sheet

---

**Upsert to Supabase**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.postgres` |
| Version | 2.5 |
| Functional role | Inserts new jobs or updates existing ones in Supabase, using `job_hash` as the unique match key. |
| Configuration | • Operation: **upsert** <br>• Schema: `public` <br>• Table: `YOUR_TABLE_NAME` (placeholder – must be changed) <br>• Matching columns: `job_hash` <br>• Column mapping (defineBelow): `salary`, `status`, `company`, `country`, `ats_type`, `job_hash`, `location`, `apply_url`, `job_title`, `work_mode`, `description`, `employment_type` <br>• `attemptToConvertTypes: false`, `convertFieldsToString: false` |
| Schema definition | Columns defined: `id` (number, removed), `job_title` (string, required), `company` (string, required), `location`, `country`, `work_mode`, `employment_type`, `apply_url`, `ats_type`, `job_hash` (string, required, used for matching), `description`, `salary`, `scraped_at` (dateTime, removed), `status`, `created_at` (dateTime, removed) |
| Error handling | `continueRegularOutput` – a failed upsert does not stop the workflow. |
| Credentials required | Supabase Postgres connection (host, port, database, user, password/SSL). |
| Edge cases | • If the table doesn't exist, the node will error (but continues). <br>• Column name mismatches between the mapping and the actual table schema cause runtime errors. <br>• The `_empty: true` sentinel item will attempt an upsert with mostly null fields; the required `job_title` and `job_hash` constraints will likely reject it, which is acceptable. |

---

**Write to Google Sheet**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.googleSheets` |
| Version | 4 |
| Functional role | Appends new rows or updates existing rows (matched on `job_hash`) in a Google Sheet. |
| Configuration | • Operation: **appendOrUpdate** <br>• Document ID: `YOUR_RESOURCE_ID_HERE` (placeholder) <br>• Sheet name: `YOUR_RESOURCE_ID_HERE` (placeholder – should be the sheet name, not the ID) <br>• Matching columns: `job_hash` <br>• Column mapping (defineBelow): `job_hash`, `Job Title`, `Company`, `Location`, `Country`, `Work Mode`, `Employment Type`, `Apply URL`, `ATS`, `Salary`, `Status`, `Description` <br>• All values reference `$('Parse and Enrich and Filter').item.json.<field>` <br>• `attemptToConvertTypes: false`, `convertFieldsToString: false` |
| Credentials required | Google Sheets OAuth2 credential with write access to the target spreadsheet. |
| Input | Items from Upsert to Supabase (same data). |
| Output | Loops back to **Loop Batches (5)** for the next batch. |
| Edge cases | • If the sheet doesn't have the exact column headers, the append will create new columns or fail. <br>• Rate limiting on the Google Sheets API for very large batches. <br>• The sentinel `_empty: true` item will be written as a row of empty cells; consider filtering it out or adding a condition node if this is undesirable. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Sticky | stickyNote | Overall workflow description, how it works, setup instructions, customization tips | — | — | ## Aggregate Multi-Source Job Boards into Centralized Database. Automate the collection, parsing, and normalization of job postings from diverse ATS platforms into a unified database. How it works: 1. Schedule daily execution using the trigger. 2. Iterate through defined ATS source configurations. 3. Fetch raw job data via HTTP requests. 4. Normalize diverse JSON structures into a standard schema. 5. Filter by domain and deduplicate entries. 6. Sync results to Supabase and Google Sheets. Setup: 1. Define your job board list in the first Code node. 2. Add Supabase credentials in the Postgres node. 3. Configure Google Sheets credentials and document ID. 4. Update the ALLOWED_DOMAINS array to match your career target. Customization: Consolidate user-specific values like API keys or table IDs in a Set node at the workflow start for easy configuration. |
| Trigger and Config | stickyNote | Labels the Source Discovery and Trigger section | — | — | ## 1. Source Discovery and Trigger. Defines target ATS sources and schedules the daily sync process. |
| Parsing and Enrichment | stickyNote | Labels the Request and Normalization section | — | — | ## 2. Request and Normalization. Executes HTTP requests and uses advanced logic to map inconsistent ATS payloads to a standard schema. |
| Destination | stickyNote | Labels the Storage and Output section | — | — | ## 3. Storage and Output. Upserts standardized job data into Supabase and updates the target Google Sheet. |
| Daily 8AM IST | scheduleTrigger | Fires workflow once daily at 08:00 | — | Company List | ## 1. Source Discovery and Trigger. Defines target ATS sources and schedules the daily sync process. |
| Company List | code | Emits one item per ATS source/company configuration | Daily 8AM IST | Loop Batches (5) | ## 1. Source Discovery and Trigger. Defines target ATS sources and schedules the daily sync process. |
| Loop Batches (5) | splitInBatches | Splits sources into batches of 5 for controlled processing | Company List | Prepare Request (output 1), Write to Google Sheet (loop back) | ## 1. Source Discovery and Trigger. Defines target ATS sources and schedules the daily sync process. |
| Prepare Request | code | Builds API URLs, headers, and pagination items per source | Loop Batches (5) | HTTP Request | ## 2. Request and Normalization. Executes HTTP requests and uses advanced logic to map inconsistent ATS payloads to a standard schema. |
| HTTP Request | httpRequest | Fetches raw data from all constructed API endpoints | Prepare Request | Parse and Enrich and Filter | ## 2. Request and Normalization. Executes HTTP requests and uses advanced logic to map inconsistent ATS payloads to a standard schema. |
| Parse and Enrich and Filter | code | Detects ATS type, normalises, enriches, filters, deduplicates job listings | HTTP Request | Upsert to Supabase | ## 2. Request and Normalization. Executes HTTP requests and uses advanced logic to map inconsistent ATS payloads to a standard schema. |
| Upsert to Supabase | postgres | Upserts deduplicated jobs into a Supabase Postgres table | Parse and Enrich and Filter | Write to Google Sheet | ## 3. Storage and Output. Upserts standardized job data into Supabase and updates the target Google Sheet. |
| Write to Google Sheet | googleSheets | Appends/updates jobs in a Google Sheet (matched on job_hash) | Upsert to Supabase | Loop Batches (5) | ## 3. Storage and Output. Upserts standardized job data into Supabase and updates the target Google Sheet. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the entire workflow in a fresh n8n instance.

---

#### Step 1 – Create the Schedule Trigger

1. Add a **Schedule Trigger** node.  
2. Set the rule to **Daily** at **08:00** (adjust timezone if your server is not IST).  
3. Name it **Daily 8AM IST**.

---

#### Step 2 – Create the Company List Code Node

1. Add a **Code** node.  
2. Name it **Company List**.  
3. Paste the full `jsCode` that defines the `sources` array (see workflow JSON). The array contains objects like `{ ats: 'greenhouse', slug: 'gitlab', company: 'GitLab' }` for each company/ATS.  
4. Ensure the code ends with `return sources.map(c => ({ json: c }));`.

---

#### Step 3 – Add the Split In Batches Node

1. Add a **Split In Batches** node.  
2. Name it **Loop Batches (5)**.  
3. Set **Batch Size** to `5`.  
4. Connect **Daily 8AM IST → Company List**.  
5. Connect **Company List → Loop Batches (5)** (input).  
6. Connect **Loop Batches (5) output 1** → **Prepare Request** (the next node).  
7. The loop-back connection will be made in Step 10.

---

#### Step 4 – Create the Prepare Request Code Node

1. Add a **Code** node.  
2. Name it **Prepare Request**.  
3. Paste the `jsCode` that builds `_url`, `_headers`, and `_route` for each source, including pagination logic for SmartRecruiters (5 pages × 100 offset) and Himalayas (10 pages × 50).  
4. The code iterates `$input.all()` and produces one item per page per source.  
5. No external credentials are needed.

---

#### Step 5 – Add the HTTP Request Node

1. Add an **HTTP Request** node.  
2. Name it **HTTP Request**.  
3. Set **Method** to `GET`.  
4. Set **URL** to `={{ $json._url }}`.  
5. Enable **Send Headers** and choose **Specify Headers** → `json`.  
6. Set **JSON Headers** to `={{ JSON.stringify($json._headers) }}`.  
7. Under **Options**, set **Timeout** to `30000` ms.  
8. Under **Options → Batching**, set **Batch Size** to `5`.  
9. Set **On Error** to `Continue Regular Output`.  
10. Connect **Prepare Request → HTTP Request**.

---

#### Step 6 – Create the Parse and Enrich and Filter Code Node

1. Add a **Code** node.  
2. Name it **Parse and Enrich and Filter**.  
3. Paste the full `jsCode` (≈300 lines) that includes:
   - Helper functions: `str`, `parseSalary`, `fmt`, `parseDate`, `getCountry`, `getWorkMode`, `cleanLoc`, `normType`, `cleanDesc`, `makeHash`, `isTitleAllowed`, `shouldKeep`, `buildOutput`, `detectATS`, `getCompanyFromResp`.  
   - The `ALLOWED_DOMAINS` array of title keywords.  
   - A loop over `$input.all()` that uses `pairedItem` to match HTTP responses back to the configuration from `$('Prepare Request').all()`.  
   - A large `switch` on the detected ATS with per-platform mapping.  
   - A final `return allJobs.length > 0 ? allJobs : [{ json: { _empty: true } }];` statement.  
4. Connect **HTTP Request → Parse and Enrich and Filter**.

---

#### Step 7 – Add the Upsert to Supabase (Postgres) Node

1. Add a **Postgres** node.  
2. Name it **Upsert to Supabase**.  
3. Create or select a **Postgres credential** pointing to your Supabase project (host, port `5432`, database, user, password, SSL required).  
4. Set **Operation** to `Upsert`.  
5. Set **Schema** to `public`.  
6. Set **Table** to your target table name (replace `YOUR_TABLE_NAME`).  
7. Set **Mapping Mode** to `Define Below`.  
8. Map columns as follows:

   | Column | Expression |
   |---|---|
   | `job_title` | `={{ $json.job_title }}` |
   | `company` | `={{ $json.company }}` |
   | `location` | `={{ $json.location }}` |
   | `country` | `={{ $json.country }}` |
   | `work_mode` | `={{ $json.work_mode }}` |
   | `employment_type` | `={{ $json.employment_type }}` |
   | `apply_url` | `={{ $json.apply_url }}` |
   | `ats_type` | `={{ $json.ats_type }}` |
   | `job_hash` | `={{ $json.job_hash }}` |
   | `description` | `={{ $json.description }}` |
   | `salary` | `={{ $json.salary }}` |
   | `status` | `={{ $json.status }}` |

9. Set **Matching Columns** to `job_hash`.  
10. Set **On Error** to `Continue Regular Output`.  
11. Connect **Parse and Enrich and Filter → Upsert to Supabase**.

---

#### Step 8 – Add the Write to Google Sheet Node

1. Add a **Google Sheets** node.  
2. Name it **Write to Google Sheet**.  
3. Create or select a **Google Sheets OAuth2 credential** with write access.  
4. Set **Operation** to `Append or Update`.  
5. Set **Document ID** to your spreadsheet ID (replace `YOUR_RESOURCE_ID_HERE`).  
6. Set **Sheet Name** to the target sheet name or ID (replace `YOUR_RESOURCE_ID_HERE`).  
7. Set **Mapping Mode** to `Define Below`.  
8. Map columns:

   | Column | Expression |
   |---|---|
   | `job_hash` | `={{ $('Parse and Enrich and Filter').item.json.job_hash }}` |
   | `Job Title` | `={{ $('Parse and Enrich and Filter').item.json.job_title }}` |
   | `Company` | `={{ $('Parse and Enrich and Filter').item.json.company }}` |
   | `Location` | `={{ $('Parse and Enrich and Filter').item.json.location }}` |
   | `Country` | `={{ $('Parse and Enrich and Filter').item.json.country }}` |
   | `Work Mode` | `={{ $('Parse and Enrich and Filter').item.json.work_mode }}` |
   | `Employment Type` | `={{ $('Parse and Enrich and Filter').item.json.employment_type }}` |
   | `Apply URL` | `={{ $('Parse and Enrich and Filter').item.json.apply_url }}` |
   | `ATS` | `={{ $('Parse and Enrich and Filter').item.json.ats_type }}` |
   | `Salary` | `={{ $('Parse and Enrich and Filter').item.json.salary }}` |
   | `Status` | `={{ $('Parse and Enrich and Filter').item.json.status }}` |
   | `Description` | `={{ $('Parse and Enrich and Filter').item.json.description }}` |

9. Set **Matching Columns** to `job_hash`.  
10. Ensure **Attempt to Convert Types** and **Convert Fields to String** are both off.  
11. Connect **Upsert to Supabase → Write to Google Sheet**.

---

#### Step 9 – Close the Loop

1. Connect **Write to Google Sheet → Loop Batches (5)** (this loops back to process the next batch of 5 sources).  
2. The **Loop Batches (5)** output 0 (done) is not connected; the workflow ends naturally after the last batch.

---

#### Step 10 – Add Sticky Notes (Optional, for Documentation)

1. Add a **Sticky Note** named **Main Sticky** at the top-left. Set its content to the full workflow description (purpose, how it works, setup, customization). Set color to `2`.  
2. Add a **Sticky Note** named **Trigger and Config** covering the first three nodes. Set its content to `## 1. Source Discovery and Trigger. Defines target ATS sources and schedules the daily sync process.` Color `7`.  
3. Add a **Sticky Note** named **Parsing and Enrichment** covering Prepare Request, HTTP Request, and Parse and Enrich and Filter. Set its content to `## 2. Request and Normalization. Executes HTTP requests and uses advanced logic to map inconsistent ATS payloads to a standard schema.` Color `7`.  
4. Add a **Sticky Note** named **Destination** covering Upsert to Supabase and Write to Google Sheet. Set its content to `## 3. Storage and Output. Upserts standardized job data into Supabase and updates the target Google Sheet.` Color `7`.

---

#### Step 11 – Final Validation Checklist

| Check | Detail |
|---|---|
| Supabase table | Exists with columns: `id` (serial PK), `job_title`, `company`, `location`, `country`, `work_mode`, `employment_type`, `apply_url`, `ats_type`, `job_hash` (unique), `description`, `salary`, `scraped_at`, `status`, `created_at`. |
| Google Sheet | Has a first row with headers exactly: `job_hash`, `Job Title`, `Company`, `Location`, `Country`, `Work Mode`, `Employment Type`, `Apply URL`, `ATS`, `Salary`, `Status`, `Description`. |
| ALLOWED_DOMAINS | Edit the array inside **Parse and Enrich and Filter** to include the job-title keywords relevant to your use case. |
| Geography filter | `shouldKeep()` keeps jobs where `country === 'India'` or `work_mode === 'Remote'`. Adjust this function if you need other regions. |
| Pagination limits | SmartRecruiters: 5 pages × 100 (500 max). Himalayas: 10 pages × 50 (500 max). Change loops in **Prepare Request** to adjust. |
| Credentials | Supabase Postgres credential and Google Sheets OAuth2 credential both configured and tested. |
| Timezone | Verify the n8n server timezone matches IST or adjust the Schedule Trigger hour accordingly. |
| Empty runs | The Parse node returns `{ _empty: true }` when no jobs pass filters. Ensure your Postgres and Sheets nodes handle this gracefully (the `_empty` item will likely fail the `job_title` NOT NULL constraint, which is acceptable since it won't insert). |

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| All ATS endpoints used are **publicly accessible** and require no API keys. | Applies to Greenhouse, Lever, Ashby, Workable, SmartRecruiters, RemoteOK, Remotive, Himalayas, Arbeitnow, Jobicy. |
| The workflow processes sources in **batches of 5** to avoid overwhelming remote APIs and to keep memory usage stable. | Configured in Loop Batches (5) node. |
| Lever and RemoteOK return **bare JSON arrays**, which n8n auto-splits into individual items. The Parse node uses `pairedItem` to match each split item back to its configuration. | Critical for correct company attribution. |
| The `ALLOWED_DOMAINS` list focuses on product, data, design, engineering, and sales roles. Edit it to match your recruiting focus. | Inside Parse and Enrich and Filter code. |
| The geography filter currently keeps only **India-located** or **Remote** jobs. Modify `shouldKeep()` to include other countries. | Inside Parse and Enrich and Filter code. |
| Salary parsing supports **USD, INR, GBP, EUR** and formats Indian numbers with L (lakh) and Cr (crore) suffixes. | Inside Parse and Enrich and Filter code. |
| The `job_hash` is a deterministic hash of `ats + company + title + location` (case-insensitive, trimmed). It serves as the deduplication key in both Supabase and Google Sheets. | Inside Parse and Enrich and Filter code. |
| If a source API changes its response structure, the `detectATS()` function may mis-classify it, causing the parser to skip that source. Check n8n execution logs for `"cannot detect ATS"` messages. | Inside Parse and Enrich and Filter code. |
| The Google Sheets node references `$('Parse and Enrich and Filter').item.json` for each column. This works because both the Supabase and Sheets nodes receive items in the same order from the Parse node. | Google Sheets column mapping. |
| To add a new ATS: (1) add an entry to the `sources` array in **Company List**, (2) add a URL template in **Prepare Request**, (3) add a parser `case` block in **Parse and Enrich and Filter**. | Extensibility pattern. |
| The workflow is self-contained and does not call any sub-workflows. | — |
| The `html_json_ld` ATS type is currently defined in the URL map but no sources use it in the default `sources` array. It is a placeholder for future career-page scraping via JSON-LD extraction. | Prepare Request code. |