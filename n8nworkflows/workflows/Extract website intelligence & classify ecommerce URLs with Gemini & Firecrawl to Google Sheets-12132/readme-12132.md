Extract website intelligence & classify ecommerce URLs with Gemini & Firecrawl to Google Sheets

https://n8nworkflows.xyz/workflows/extract-website-intelligence---classify-ecommerce-urls-with-gemini---firecrawl-to-google-sheets-12132


# Extract website intelligence & classify ecommerce URLs with Gemini & Firecrawl to Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow takes a submitted website URL, scrapes and cleans its homepage HTML, uses **Google Gemini** to extract structured company intelligence, writes it to **Google Sheets**, then uses **Firecrawl** to map the site and retrieve internal URLs. Finally, it classifies URLs (e.g., category/product/other) using **Gemini**, and appends the results into dedicated Sheets tabs.

**Typical use cases:**
- Enriching a lead list with company metadata (industry, description, etc.)
- Building an ecommerce URL inventory (categories/products/other pages)
- Automated site mapping + URL classification at scale

### 1.1 Input Reception & Web Scraping
Receives a form submission containing a website URL, fetches the site, and extracts HTML for downstream processing.

### 1.2 HTML Cleanup & Company Intelligence Extraction (Gemini)
Cleans extracted HTML and asks Gemini to output structured JSON representing company info.

### 1.3 Persist Company Intelligence to Google Sheets
Parses Gemini’s JSON output and updates a “Domain Scraper” sheet.

### 1.4 Website Mapping (Firecrawl) & URL Normalization
Maps the website via Firecrawl, parses URL metadata, and produces an array of URLs suitable for batching.

### 1.5 URL Classification (Gemini) & Writing Results to Sheets
Batches URLs, sends each batch to Gemini for classification, splits results into Categories / Products / Others, appends each to its respective Google Sheets tab, and loops until all batches are processed.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Web Scraping
**Overview:** Receives a website URL from an n8n Form and downloads the site HTML for extraction. This block is tolerant of failures (continue-on-fail) in the HTTP and HTML steps.  
**Nodes involved:** Form Submission → Scrape Website URL → Extract HTML

#### Node: **Form Submission**
- **Type / role:** `formTrigger` — entry point; starts workflow when an n8n Form is submitted.
- **Configuration (interpreted):** Uses an internal webhook (auto-managed by n8n) to receive form fields.
- **Inputs / outputs:** No inputs (trigger). Output is the submitted form data (likely including a URL field).
- **Edge cases / failures:**
  - Missing/invalid URL field (later nodes may fail or extract empty HTML).
  - Multiple fields: expressions in later nodes must reference the correct field name.

#### Node: **Scrape Website URL**
- **Type / role:** `httpRequest` — fetches the provided website URL.
- **Configuration choices:**
  - **Retry on fail:** enabled.
  - **Continue on fail:** enabled (workflow continues even if request errors).
  - Other HTTP parameters are not present in JSON (so they’re either defaults or removed from export). In a working workflow, this typically includes:
    - URL expression from the form data
    - Method: GET
    - Response: text (HTML)
- **Connections:** Input from Form Submission; output to Extract HTML.
- **Edge cases / failures:**
  - 4xx/5xx responses, SSL errors, DNS failures.
  - Cloudflare / bot protections returning challenge pages.
  - Huge HTML payloads causing memory/time issues.
  - Because **continueOnFail=true**, downstream nodes must handle missing/empty response bodies.

#### Node: **Extract HTML**
- **Type / role:** `htmlExtract` — extracts relevant content from raw HTML.
- **Configuration choices:** Not specified in JSON; typically configured with selectors to pull text, title, meta tags, etc.
- **Continue on fail:** enabled.
- **Connections:** Input from Scrape Website URL; output to Clean HTML Content.
- **Edge cases / failures:**
  - If Scrape Website URL fails, extractor may receive empty input.
  - Bad selectors can return empty fields.
  - Non-HTML responses (JSON, binary) break extraction.

---

### Block 2 — HTML Cleanup & Company Intelligence Extraction (Gemini)
**Overview:** Takes extracted HTML/text, cleans it, and uses Gemini to generate structured company information (ideally JSON).  
**Nodes involved:** Clean HTML Content → Company Info Agent → Parse JSON Data

#### Node: **Clean HTML Content**
- **Type / role:** `code` — transforms/cleans extracted HTML fields into a prompt-friendly text blob.
- **Configuration choices:** Code not included in export; typical actions:
  - Strip scripts/styles, remove excessive whitespace
  - Truncate length to fit LLM context
  - Combine title/meta/body text
- **Continue on fail:** enabled.
- **Connections:** Input from Extract HTML; output to Company Info Agent.
- **Edge cases / failures:**
  - Code exceptions (undefined fields).
  - Over-truncation leading to weak extraction.
  - If upstream extraction is empty, output may be empty and Gemini results unreliable.

#### Node: **Company Info Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — LLM call to extract company intelligence.
- **Configuration choices (interpreted):**
  - Prompts/temperature/model are not shown; but output is expected to be parseable JSON (given next node is “Parse JSON Data”).
  - Must be configured with **Google Gemini credentials** in n8n.
- **Connections:** Input from Clean HTML Content; output to Parse JSON Data.
- **Edge cases / failures:**
  - Non-JSON or partial JSON output (common LLM failure mode).
  - Safety filters / refusals.
  - Token limits if cleaned content too large.
  - Credential / API quota errors.

#### Node: **Parse JSON Data**
- **Type / role:** `code` — parses Gemini output into structured fields for Sheets update.
- **Configuration choices:** Not shown; typically:
  - `JSON.parse()` on the LLM text output
  - Validation and defaults for missing keys
- **Connections:** Input from Company Info Agent; output to Update Domain Scraper Sheet.
- **Edge cases / failures:**
  - JSON parsing errors if Gemini returns markdown, commentary, or malformed JSON.
  - Missing required keys for the Sheets mapping.

---

### Block 3 — Persist Company Intelligence to Google Sheets
**Overview:** Writes the extracted company info into a Google Sheet, then proceeds to site mapping.  
**Nodes involved:** Update Domain Scraper Sheet → Map a website and get urls

#### Node: **Update Domain Scraper Sheet**
- **Type / role:** `googleSheets` (v3) — updates an existing row or range with the parsed company intelligence.
- **Configuration choices:** Not included; typically includes:
  - Spreadsheet ID
  - Sheet/tab name (e.g., “Domain Scraper”)
  - Operation: Update / Upsert / Append (name suggests Update)
  - Field mapping from Parse JSON Data outputs
- **Connections:** Input from Parse JSON Data; output to Map a website and get urls.
- **Edge cases / failures:**
  - Google OAuth credential expired/invalid.
  - Wrong sheet name / permissions.
  - Update operation requires a row identifier; if missing, update may fail or overwrite unintended rows.

---

### Block 4 — Website Mapping (Firecrawl) & URL Normalization
**Overview:** Uses Firecrawl to map the website and generate a list of internal URLs. Then it normalizes/parses the results into an array suitable for batching.  
**Nodes involved:** Map a website and get urls → Parse URLs with MetaData → Parse Array URLs → Split in Batches

#### Node: **Map a website and get urls**
- **Type / role:** `@mendable/n8n-nodes-firecrawl.firecrawl` — Firecrawl mapping/crawling to collect URLs.
- **Configuration choices:** Parameters not shown; typically:
  - Mode: “Map” (discover URLs)
  - Start URL/domain from earlier steps
  - Limits: max pages, depth, include/exclude patterns
  - Firecrawl API key credential required
- **Connections:** Input from Update Domain Scraper Sheet; output to Parse URLs with MetaData.
- **Edge cases / failures:**
  - Firecrawl auth/quota errors.
  - Robots restrictions, timeouts, crawl limits.
  - Large sites producing huge URL lists (memory + batching considerations).

#### Node: **Parse URLs with MetaData**
- **Type / role:** `code` (v2) — reshapes Firecrawl output (often contains URL + metadata) into a consistent internal structure.
- **Configuration choices:** Not shown; common actions:
  - Extract `url`, `title`, `status`, `contentType` if present
  - Filter to same domain / remove duplicates
- **Connections:** Input from Firecrawl node; output to Parse Array URLs.
- **Edge cases / failures:**
  - Firecrawl response shape changes or missing fields.
  - Unexpected nested arrays/objects causing mapping errors.

#### Node: **Parse Array URLs**
- **Type / role:** `code` (v2) — converts the URL list into an array of items for batching (often 1 URL per item).
- **Connections:** Input from Parse URLs with MetaData; output to Split in Batches.
- **Edge cases / failures:**
  - Empty URL list leads to no batches (downstream classification won’t run).
  - Duplicate URLs causing redundant sheet entries unless deduped.

#### Node: **Split in Batches**
- **Type / role:** `splitInBatches` — iterates through URL items in manageable batch sizes.
- **Configuration choices:** Batch size not shown (default is often 1 unless changed).
- **Connections:** Input from Parse Array URLs **and** loop-back inputs from Append Categories/Products/Others; output to Categorising AI Agent.
- **Edge cases / failures:**
  - If batch size too large, Gemini prompt may exceed token limits.
  - Loop logic relies on correct “next batch” behavior; miswiring can cause infinite loops or early termination.

---

### Block 5 — URL Classification (Gemini) & Writing Results to Sheets
**Overview:** For each batch, Gemini classifies URLs; results are split into categories/products/others and appended to separate Google Sheets tabs; each append triggers the next batch.  
**Nodes involved:** Categorising AI Agent → Parse All URLs with categories → (Parse Categories → Append Categories) + (Parse Products → Append Products) + (Parse Others → Append Others) → back to Split in Batches

#### Node: **Categorising AI Agent**
- **Type / role:** Google Gemini LLM node — classifies URLs into ecommerce-relevant groups.
- **Configuration choices:** Not shown; typically includes:
  - Prompt instructing model to label each URL as `category`, `product`, or `other`
  - Output format expected to be machine-parsable (JSON)
  - Credentials: Gemini API
- **Connections:** Input from Split in Batches; output to Parse All URLs with categories.
- **Edge cases / failures:**
  - Malformed output or missing URLs in response.
  - Token limits if batching too many URLs.
  - Model may hallucinate categories or misclassify non-ecommerce pages.

#### Node: **Parse All URLs with categories**
- **Type / role:** `code` (v2) — parses Gemini response and creates three collections for downstream nodes.
- **Configuration choices:** Not shown; likely:
  - Parse JSON
  - Build arrays for categories/products/others
- **Connections:** Input from Categorising AI Agent; outputs to Parse Categories, Parse Products, Parse Others (three parallel branches).
- **Edge cases / failures:**
  - Partial parsing causing one branch to be empty.
  - If Gemini returns a single string instead of structured JSON, parsing fails and nothing gets appended.

#### Node: **Parse Categories**
- **Type / role:** `code` (v2) — formats “category” URLs into rows for Sheets.
- **Connections:** Output to Append Categories.
- **Edge cases:** Empty categories list should yield zero items (ensure code returns `[]` cleanly).

#### Node: **Append Categories**
- **Type / role:** `googleSheets` (v4.7) — appends category URL rows to a designated tab.
- **Connections:** Output loops back into Split in Batches.
- **Edge cases / failures:**
  - Duplicate entries if rerun (no dedupe unless sheet/formulas handle it).
  - Append failures due to sheet limits or permission issues.

#### Node: **Parse Products**
- **Type / role:** `code` (v2) — formats “product” URLs into rows for Sheets.
- **Connections:** Output to Append Products.

#### Node: **Append Products**
- **Type / role:** `googleSheets` (v4.7) — appends product URL rows to a designated tab.
- **Connections:** Output loops back into Split in Batches.

#### Node: **Parse Others**
- **Type / role:** `code` (v2) — formats “other” URLs into rows for Sheets.
- **Connections:** Output to Append Others.

#### Node: **Append Others**
- **Type / role:** `googleSheets` (v4.7) — appends “other” URL rows to a designated tab.
- **Connections:** Output loops back into Split in Batches.

**Looping note:**  
All three append nodes connect back to **Split in Batches**. In practice, this can cause:
- Multiple “next batch” triggers per batch (one per branch), depending on how n8n executes branches and the SplitInBatches node behavior.
- A safer pattern is usually: merge branches → single “continue” to Split in Batches, or append sequentially.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form Submission | n8n-nodes-base.formTrigger | Workflow entry via form | — | Scrape Website URL |  |
| Scrape Website URL | n8n-nodes-base.httpRequest | Download website HTML | Form Submission | Extract HTML |  |
| Extract HTML | n8n-nodes-base.htmlExtract | Extract relevant HTML/text fields | Scrape Website URL | Clean HTML Content |  |
| Clean HTML Content | n8n-nodes-base.code | Sanitize/prepare text for LLM | Extract HTML | Company Info Agent |  |
| Company Info Agent | @n8n/n8n-nodes-langchain.googleGemini | Extract company info as structured output | Clean HTML Content | Parse JSON Data |  |
| Parse JSON Data | n8n-nodes-base.code | Parse LLM output JSON into fields | Company Info Agent | Update Domain Scraper Sheet |  |
| Update Domain Scraper Sheet | n8n-nodes-base.googleSheets | Persist company intelligence | Parse JSON Data | Map a website and get urls |  |
| Map a website and get urls | @mendable/n8n-nodes-firecrawl.firecrawl | Map site and retrieve URLs | Update Domain Scraper Sheet | Parse URLs with MetaData |  |
| Parse URLs with MetaData | n8n-nodes-base.code | Normalize Firecrawl URL metadata | Map a website and get urls | Parse Array URLs |  |
| Parse Array URLs | n8n-nodes-base.code | Convert URL list to batchable items | Parse URLs with MetaData | Split in Batches |  |
| Split in Batches | n8n-nodes-base.splitInBatches | Iterate through URLs in batches | Parse Array URLs; Append Categories; Append Products; Append Others | Categorising AI Agent |  |
| Categorising AI Agent | @n8n/n8n-nodes-langchain.googleGemini | Classify URLs (category/product/other) | Split in Batches | Parse All URLs with categories |  |
| Parse All URLs with categories | n8n-nodes-base.code | Split classification results into 3 streams | Categorising AI Agent | Parse Categories; Parse Products; Parse Others |  |
| Parse Categories | n8n-nodes-base.code | Format category rows | Parse All URLs with categories | Append Categories |  |
| Append Categories | n8n-nodes-base.googleSheets | Append category URLs to sheet | Parse Categories | Split in Batches |  |
| Parse Products | n8n-nodes-base.code | Format product rows | Parse All URLs with categories | Append Products |  |
| Append Products | n8n-nodes-base.googleSheets | Append product URLs to sheet | Parse Products | Split in Batches |  |
| Parse Others | n8n-nodes-base.code | Format other rows | Parse All URLs with categories | Append Others |  |
| Append Others | n8n-nodes-base.googleSheets | Append other URLs to sheet | Parse Others | Split in Batches |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |

**Sticky note content:** all sticky notes have empty content in the provided JSON, so there are no comments/links to propagate.

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add node: **Form Trigger** (`Form Submission`)  
   2) Create a form field such as `website_url` (exact name must match what you reference later).  
   3) Activate the form and copy its URL if needed for testing.

2. **Fetch the Website**
   1) Add node: **HTTP Request** (`Scrape Website URL`)  
   2) Set **Method**: `GET`  
   3) Set **URL** to the form value (example expression): `{{$json.website_url}}`  
   4) Set **Response Format**: `String` (HTML text)  
   5) Enable **Retry on Fail** and **Continue On Fail** (to match the workflow behavior).

3. **Extract HTML Signals**
   1) Add node: **HTML Extract** (`Extract HTML`)  
   2) Set **HTML** input to the HTTP response body (commonly `{{$json.body}}` depending on your HTTP node settings).  
   3) Add selectors you need (title, meta description, body text, etc.).  
   4) Enable **Continue On Fail**.

4. **Clean/Prepare Text**
   1) Add node: **Code** (`Clean HTML Content`)  
   2) Implement cleaning logic (strip tags/noise, normalize whitespace, truncate).  
   3) Output a field like `cleanText` to feed Gemini.

5. **Extract Company Info with Gemini**
   1) Add node: **Google Gemini** (`Company Info Agent`)  
   2) Configure **credentials**: Google/Gemini API key (or OAuth depending on node support).  
   3) Prompt: ask for company info **as strict JSON** (important for parsing), e.g. keys like:
      - `domain`, `company_name`, `description`, `industry`, `country`, `emails`, `social_links`, etc.
   4) Ensure output is plain JSON (no markdown fences).

6. **Parse Company JSON**
   1) Add node: **Code** (`Parse JSON Data`)  
   2) Parse Gemini text output with `JSON.parse()` and map to flat fields for Google Sheets.  
   3) Add guardrails (try/catch; defaults if missing).

7. **Write Company Info to Google Sheets**
   1) Add node: **Google Sheets** (`Update Domain Scraper Sheet`)  
   2) Configure Google credentials (OAuth2).  
   3) Choose Spreadsheet + Tab (e.g., “Domain Scraper”).  
   4) Choose operation consistent with your needs:
      - If you have a unique key (domain), use **Upsert** (if available) or search+update pattern.
      - Otherwise use **Append**.
   5) Map parsed fields to columns.

8. **Map the Website with Firecrawl**
   1) Add node: **Firecrawl** (`Map a website and get urls`)  
   2) Configure Firecrawl API credentials.  
   3) Set start URL/domain from earlier steps (e.g., `{{$json.domain}}` or the submitted URL).  
   4) Configure crawl/map limits (depth, max URLs) to control cost and size.

9. **Normalize Firecrawl Output**
   1) Add node: **Code** (`Parse URLs with MetaData`)  
   2) Extract the list of URLs (and optional metadata), filter to same domain, dedupe.

10. **Prepare Items for Batching**
   1) Add node: **Code** (`Parse Array URLs`)  
   2) Convert URL array into n8n items: one item per URL or per small group, depending on classification prompt design.

11. **Batch Processing Loop**
   1) Add node: **Split In Batches** (`Split in Batches`)  
   2) Set batch size (start with 10–50; adjust to avoid Gemini token limits).  
   3) Connect `Parse Array URLs` → `Split in Batches`.

12. **Classify URLs with Gemini**
   1) Add node: **Google Gemini** (`Categorising AI Agent`)  
   2) Prompt Gemini to label each URL as one of: `category`, `product`, `other` and return strict JSON:
      - Include the original URL and the assigned type.
   3) Connect `Split in Batches` → `Categorising AI Agent`.

13. **Parse Classification Result into 3 Streams**
   1) Add node: **Code** (`Parse All URLs with categories`)  
   2) Parse the JSON, split into three arrays (categories/products/others).  
   3) Output three separate streams (or three outputs via separate nodes as in this workflow).

14. **Format + Append Each Stream to Sheets**
   - Categories:
     1) **Code** (`Parse Categories`) to format rows  
     2) **Google Sheets** (`Append Categories`) set to Append into a “Categories” tab
   - Products:
     1) **Code** (`Parse Products`)  
     2) **Google Sheets** (`Append Products`) into a “Products” tab
   - Others:
     1) **Code** (`Parse Others`)  
     2) **Google Sheets** (`Append Others`) into an “Others” tab

15. **Loop Back for Next Batch**
   - Connect each Append node back to **Split in Batches** to request the next batch.
   - Practical recommendation when rebuilding: use a **Merge** (wait for all branches) and then loop back once, to avoid multiple “continue” signals per batch.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The provided workflow includes multiple Sticky Note nodes, but their content is empty. | No additional annotations/links were included in the JSON export. |
| Many nodes have empty parameter blocks in the export; to reproduce behavior you must re-enter prompts, selectors, sheet IDs, and field mappings. | Applies to HTTP Request, HTML Extract, Code nodes, Gemini nodes, Firecrawl, and Google Sheets nodes. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.