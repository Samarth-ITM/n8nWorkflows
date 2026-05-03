Discover LinkedIn leads and draft outreach using Apify, Google Sheets, and Gemini

https://n8nworkflows.xyz/workflows/discover-linkedin-leads-and-draft-outreach-using-apify--google-sheets--and-gemini-12459


# Discover LinkedIn leads and draft outreach using Apify, Google Sheets, and Gemini

## 1. Workflow Overview

**Purpose:** This workflow discovers potential LinkedIn leads using Apify searches, stores them in Google Sheets, then enriches each profile (via scraping) and uses **Google Gemini** to draft a **connection message** and a **follow-up message**—all saved back to Google Sheets.

**Typical use cases**
- Lead sourcing for outbound sales/recruiting
- Building a searchable leads database in Google Sheets
- Generating first-touch and follow-up outreach drafts at scale (human-in-the-loop sending)

### 1.1 Lead Discovery (Query → Apify Search → Store)
Generates LinkedIn search queries, iterates through them, calls Apify to retrieve results, flattens the dataset, and saves leads to Google Sheets.

### 1.2 Lead Enrichment + Message Drafting (Sheet → Scrape → Gemini → Update Sheet)
Loads unenriched leads from Google Sheets, loops over each profile, scrapes LinkedIn profile data (via Apify or similar), generates outreach text with Gemini, saves enrichment + messages, and marks the lead as enriched.

**Entry points:** There is no explicit trigger node in the provided JSON (e.g., Manual Trigger/Cron/Webhook). Practically, you would add a trigger or run blocks manually.

---

## 2. Block-by-Block Analysis

### Block A — Generate Search Queries and Discover Leads in Apify
**Overview:** Creates one or more LinkedIn search queries, iterates over them in batches, searches via Apify, transforms results into a tabular format, and persists them to Google Sheets.

**Nodes involved**
- Generate LinkedIn Search Queries (Code)
- Loop Over Items (Split in Batches)
- Search in Apify (HTTP Request)
- Flatten Apify Results (Code)
- Save in Database (Google Sheets)

#### 1) Generate LinkedIn Search Queries
- **Type / role:** `Code` node — constructs an array of search query items to feed the loop.
- **Configuration (interpreted):** Not provided in JSON. Expected to output multiple items, each containing parameters used by **Search in Apify** (e.g., keywords, location, title, page limits).
- **Key variables/expressions:** Not visible; commonly outputs fields like `query`, `searchUrl`, or Apify actor input payload.
- **Connections:** Output → **Loop Over Items**
- **Edge cases / failures:**
  - Producing zero items will stop the discovery pipeline.
  - Malformed fields (missing required keys) will cause the HTTP request node to fail or return empty results.

#### 2) Loop Over Items
- **Type / role:** `SplitInBatches` — batch/iterate through generated queries.
- **Configuration (interpreted):** Not provided; defaults typically process a batch size (often 1).
- **Connections:**
  - Output (index 0) → **Search in Apify**
  - Output (index 1) → loops back to **Loop Over Items** (this is the “continue” output to fetch next batch)
- **Edge cases / failures:**
  - Incorrect batch size could overload Apify / rate limit.
  - If the “continue” loop is misconfigured, it can create infinite loops or stop after first batch depending on settings.

#### 3) Search in Apify
- **Type / role:** `HTTP Request` — calls Apify API to perform LinkedIn search/scrape.
- **Configuration (interpreted):** Not provided; expected:
  - URL like `https://api.apify.com/v2/...`
  - Auth via Apify token (header/query param)
  - Payload driven by fields from the current batch item
- **Connections:** Output → **Flatten Apify Results**
- **Likely outputs:** Raw Apify run info or dataset items.
- **Edge cases / failures:**
  - 401/403 if token invalid/expired.
  - 429 rate limit; 5xx from Apify.
  - Long runs: if using actor runs, you may need polling/waiting; a single HTTP call may only start a run rather than return results.

#### 4) Flatten Apify Results
- **Type / role:** `Code` — transforms nested Apify response/dataset into “one item per lead” with flat fields suitable for Sheets.
- **Configuration (interpreted):** Not provided; expected mapping like:
  - name, headline, company, location, profileUrl, etc.
- **Connections:** Output → **Save in Database**
- **Edge cases / failures:**
  - Apify schema changes or missing fields can break mapping.
  - If Apify returns empty dataset, this node should output zero items (Sheets insert would do nothing).

#### 5) Save in Database
- **Type / role:** `Google Sheets` — persists discovered leads into a spreadsheet (acting as the “database”).
- **Configuration (interpreted):** Not provided; expected:
  - Spreadsheet + Sheet tab
  - Append rows (or upsert-like behavior if configured)
  - Column mapping from flattened fields
- **Connections:** No outgoing connections shown (end of Block A).
- **Edge cases / failures:**
  - Google OAuth credential issues; permission denied to the sheet.
  - Column mismatch: missing required columns or wrong headers.
  - Duplicate leads: if no dedupe key (e.g., profile URL), sheet can accumulate duplicates.

---

### Block B — Load Unenriched Leads and Enrich Each Profile
**Overview:** Pulls leads that are not yet enriched, loops through them, scrapes profile data, generates two message drafts with Gemini, updates Google Sheets, and marks each lead as enriched.

**Nodes involved**
- Load Unenriched Profiles (Google Sheets)
- Process Each Profile (Split in Batches)
- Scrape LinkedIn Profile Data (HTTP Request)
- Generate Connection Message (Google Gemini)
- Save Enriched Lead Data (Google Sheets)
- Generate Follow-Up Message (Google Gemini)
- Update Follow-Up in Sheet (Google Sheets)
- Mark Profile as Enriched (Google Sheets)

#### 6) Load Unenriched Profiles
- **Type / role:** `Google Sheets` — reads leads needing enrichment.
- **Configuration (interpreted):** Not provided; expected:
  - “Read” operation with filtering, or reading all and filtering in-node/elsewhere
  - Typically uses a column like `enriched = FALSE` or blank
- **Connections:** Output → **Process Each Profile**
- **Edge cases / failures:**
  - If filtering is not implemented, you may re-enrich already enriched leads.
  - Large sheets can cause long read times/timeouts.

#### 7) Process Each Profile
- **Type / role:** `SplitInBatches` — processes leads one at a time (or in small batches) to control rate and cost.
- **Configuration (interpreted):** Not provided.
- **Connections:**
  - Output (index 1) → **Scrape LinkedIn Profile Data**
  - Output (index 0) is unused (connected to nothing), which is unusual: typically index 0 is the “batch” output and index 1 is “no items left”. Here, the workflow uses index 1 to proceed.
- **Edge cases / failures:**
  - **Potential miswiring:** In many n8n setups, the first output is the “current batch” and the second is “done”. If that’s the case here, the workflow would only continue when there are *no items*, effectively doing nothing. Verify output semantics in your n8n version.
  - If batch size is too high, scraping + Gemini calls may hit rate limits or cost spikes.

#### 8) Scrape LinkedIn Profile Data
- **Type / role:** `HTTP Request` — fetches detailed profile info (commonly via Apify actor or a scraping endpoint).
- **Configuration (interpreted):** Not provided; expected to use the profile URL/id from the sheet row.
- **Connections:** Output → **Generate Connection Message**
- **Edge cases / failures:**
  - LinkedIn scraping is brittle: blocked requests, captcha, partial data.
  - Apify actor may require waiting/polling; a single call might not return dataset items immediately.

#### 9) Generate Connection Message
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — drafts an initial LinkedIn connection message based on scraped profile data.
- **Configuration (interpreted):** Not provided; expected:
  - Model selection (Gemini model)
  - Prompt using fields like name, role, company, mutual interests, etc.
- **Connections:** Output → **Save Enriched Lead Data**
- **Version-specific requirements:**
  - Requires the LangChain Gemini node package available in the n8n instance.
  - Requires Google AI/Gemini credentials configured in n8n.
- **Edge cases / failures:**
  - Prompt variables missing → low-quality or generic output.
  - Safety filters/refusals depending on prompt content.
  - Quota/rate limiting from Google.

#### 10) Save Enriched Lead Data
- **Type / role:** `Google Sheets` — writes scraped enrichment fields + connection message back to the lead row (or another tab).
- **Configuration (interpreted):** Not provided; expected:
  - Update row by ID / row number / unique key (profile URL)
- **Connections:** Output → **Generate Follow-Up Message**
- **Edge cases / failures:**
  - If not updating deterministically (no key), you may overwrite wrong rows.
  - Column header mismatch.

#### 11) Generate Follow-Up Message
- **Type / role:** `Google Gemini` — drafts a follow-up message.
- **Configuration (interpreted):** Not provided; expected prompt includes previous message and/or enriched profile context.
- **Connections:** Output → **Update Follow-Up in Sheet**
- **Edge cases / failures:** Same as Node 9 (quota, safety, missing variables).

#### 12) Update Follow-Up in Sheet
- **Type / role:** `Google Sheets` — updates the sheet row with follow-up content.
- **Configuration (interpreted):** Not provided; expected update-by-key.
- **Connections:** Output → **Mark Profile as Enriched**
- **Edge cases / failures:** Row identification issues; partial updates leaving inconsistent state.

#### 13) Mark Profile as Enriched
- **Type / role:** `Google Sheets` — sets a flag (e.g., `enriched=true`, timestamp) to prevent reprocessing.
- **Configuration (interpreted):** Not provided.
- **Connections:** Output → **Process Each Profile** (loops to process next lead)
- **Edge cases / failures:**
  - If the flag update fails, the same lead will be reprocessed on the next run.
  - Loop correctness depends on SplitInBatches wiring/settings.

---

### Block C — Sticky Notes / Annotations
**Overview:** The workflow contains multiple sticky note nodes, but their contents are empty in the provided JSON. They do not affect execution.

**Nodes involved**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note4
- Sticky Note5
- Sticky Note6

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Search in Apify | HTTP Request | Call Apify API to run LinkedIn search/scrape | Loop Over Items | Flatten Apify Results |  |
| Flatten Apify Results | Code | Convert Apify response into flat lead rows | Search in Apify | Save in Database |  |
| Save in Database | Google Sheets | Append/store discovered leads | Flatten Apify Results |  |  |
| Sticky Note | Sticky Note | Annotation (empty) |  |  |  |
| Generate LinkedIn Search Queries | Code | Build search query items for Apify |  | Loop Over Items |  |
| Loop Over Items | Split in Batches | Iterate through search queries | Generate LinkedIn Search Queries; Loop Over Items (self-loop) | Search in Apify; Loop Over Items (self-loop) |  |
| Sticky Note1 | Sticky Note | Annotation (empty) |  |  |  |
| Sticky Note2 | Sticky Note | Annotation (empty) |  |  |  |
| Sticky Note4 | Sticky Note | Annotation (empty) |  |  |  |
| Sticky Note5 | Sticky Note | Annotation (empty) |  |  |  |
| Sticky Note6 | Sticky Note | Annotation (empty) |  |  |  |
| Load Unenriched Profiles | Google Sheets | Read leads pending enrichment |  | Process Each Profile |  |
| Process Each Profile | Split in Batches | Iterate through unenriched profiles | Load Unenriched Profiles; Mark Profile as Enriched | Scrape LinkedIn Profile Data |  |
| Scrape LinkedIn Profile Data | HTTP Request | Retrieve detailed profile data (likely via Apify) | Process Each Profile | Generate Connection Message |  |
| Generate Connection Message | Google Gemini (LangChain) | Draft connection invite text | Scrape LinkedIn Profile Data | Save Enriched Lead Data |  |
| Save Enriched Lead Data | Google Sheets | Update sheet with enrichment + connection message | Generate Connection Message | Generate Follow-Up Message |  |
| Generate Follow-Up Message | Google Gemini (LangChain) | Draft follow-up message | Save Enriched Lead Data | Update Follow-Up in Sheet |  |
| Update Follow-Up in Sheet | Google Sheets | Update sheet with follow-up text | Generate Follow-Up Message | Mark Profile as Enriched |  |
| Mark Profile as Enriched | Google Sheets | Set enriched flag/status | Update Follow-Up in Sheet | Process Each Profile |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.
2) **Add a trigger** (not included in JSON), choose one:
   - *Manual Trigger* for testing, or
   - *Cron* to run daily/weekly, or
   - *Webhook* if driven externally.

### A) Lead discovery chain
3) Add **Code** node: **Generate LinkedIn Search Queries**
   - Output one item per search query.
   - Each item should include the data your Apify search expects (e.g., `keywords`, `location`, `maxResults`, or a full `actorInput` object).
4) Add **Split in Batches**: **Loop Over Items**
   - Set batch size (commonly `1` to control rate).
   - Connect **Generate LinkedIn Search Queries → Loop Over Items**.
5) Add **HTTP Request**: **Search in Apify**
   - Configure Apify authentication (token).
   - Configure request to either:
     - Start an Apify actor run with input based on the current item, and/or
     - Fetch dataset items.
   - Connect **Loop Over Items (batch output) → Search in Apify**.
   - Connect **Loop Over Items (continue output) → Loop Over Items** to iterate.
6) Add **Code**: **Flatten Apify Results**
   - Map the Apify response into flat lead objects with consistent keys (e.g., `fullName`, `headline`, `company`, `location`, `profileUrl`).
   - Connect **Search in Apify → Flatten Apify Results**.
7) Add **Google Sheets**: **Save in Database**
   - Set Google Sheets credential (Google OAuth2).
   - Choose Spreadsheet + Sheet.
   - Operation: Append rows (or Update/Upsert if you implement dedupe).
   - Map fields from flattened output into columns.
   - Connect **Flatten Apify Results → Save in Database**.

### B) Enrichment + messaging chain
8) Add **Google Sheets**: **Load Unenriched Profiles**
   - Operation: Read rows.
   - Implement a filter strategy:
     - Preferred: a column like `enriched` and only read rows where it’s blank/false (implementation depends on your node approach; some users read all then filter via IF node).
9) Add **Split in Batches**: **Process Each Profile**
   - Batch size: `1` recommended (scraping + LLM).
   - Connect **Load Unenriched Profiles → Process Each Profile**.
   - Ensure you are using the correct SplitInBatches output for “current batch” in your n8n version (verify by testing).
10) Add **HTTP Request**: **Scrape LinkedIn Profile Data**
   - Use the lead’s `profileUrl` (or identifier) from the current item.
   - Call Apify actor / endpoint that returns detailed profile data.
   - Connect **Process Each Profile → Scrape LinkedIn Profile Data**.
11) Add **Google Gemini (LangChain)**: **Generate Connection Message**
   - Configure Gemini credentials.
   - Choose a model and provide a prompt that uses scraped fields (name/role/company) and produces a short LinkedIn invite message.
   - Connect **Scrape LinkedIn Profile Data → Generate Connection Message**.
12) Add **Google Sheets**: **Save Enriched Lead Data**
   - Operation: Update row (use a stable key: row number, or a unique `profileUrl` column).
   - Write enrichment fields + connection message.
   - Connect **Generate Connection Message → Save Enriched Lead Data**.
13) Add **Google Gemini (LangChain)**: **Generate Follow-Up Message**
   - Prompt for a follow-up message, optionally referencing the connection message and enriched info.
   - Connect **Save Enriched Lead Data → Generate Follow-Up Message**.
14) Add **Google Sheets**: **Update Follow-Up in Sheet**
   - Update the same row with the follow-up text.
   - Connect **Generate Follow-Up Message → Update Follow-Up in Sheet**.
15) Add **Google Sheets**: **Mark Profile as Enriched**
   - Update the row to set `enriched=true` and optionally `enrichedAt=NOW()`.
   - Connect **Update Follow-Up in Sheet → Mark Profile as Enriched**.
16) **Loop to next profile**
   - Connect **Mark Profile as Enriched → Process Each Profile** to continue batching until complete.

### Credentials required
- **Apify:** API token used by HTTP Request node(s).
- **Google Sheets:** Google OAuth2 with edit access to the spreadsheet.
- **Google Gemini:** Gemini/Google AI credentials configured for the LangChain Gemini node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer (applies to the entire workflow) |
| Sticky notes are present but empty. | No additional operational guidance embedded in the workflow canvas |

