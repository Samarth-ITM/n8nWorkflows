Generate scored B2B leads from Google Maps websites to Google Sheets

https://n8nworkflows.xyz/workflows/generate-scored-b2b-leads-from-google-maps-websites-to-google-sheets-11924


# Generate scored B2B leads from Google Maps websites to Google Sheets

## 1. Workflow Overview

**Purpose:** This workflow generates **scored B2B leads** by searching Google Maps (Google Places API v1) with user-provided queries, collecting matching businesses, scraping their websites (homepage + contact-related pages) to extract emails, scoring email quality, keeping the best email per business, deduplicating results, and exporting leads to **Google Sheets**.

**Typical use cases:**
- Prospecting: “Dentist New York”, “Plumber London”, “SaaS agency Berlin”
- Building outbound lists with validated website + email signals
- Producing a spreadsheet-ready lead list with basic quality scoring

### 1.1 Logical Blocks
1. **Input & Query Loop**: manual start + iterate over search queries.
2. **Google Maps Search + Pagination**: call Places API searchText and accumulate results across pages.
3. **Business Filtering & Field Preparation**: keep OPERATIONAL businesses, split into items, extract core fields.
4. **Website + Contact Page Scraping**: fetch homepage, find contact links, fetch contact pages, combine HTML.
5. **Email Extraction & Filtering**: regex-extract emails, filter out junk patterns.
6. **Email Quality Scoring & Best Selection**: group per business, score, sort, keep best, dedupe emails.
7. **Merge, Deduplicate Businesses, Export**: merge enriched email data with business stream, dedupe businesses, append to Google Sheets, loop back for next query batch.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Query Loop
**Overview:** Starts the workflow manually and iterates through a list of search queries using batching mechanics.

**Nodes involved:**
- **Run workflow**
- **Loop over queries**

#### Node: Run workflow
- **Type / role:** Manual Trigger; entry point.
- **Config choices:** No parameters; run-on-demand only.
- **Connections:** Output → **Loop over queries**.
- **Edge cases:** None (manual start only).

#### Node: Loop over queries
- **Type / role:** Split In Batches; iterates over query items.
- **Config choices:** Default options (batch size not explicitly set; relies on node default).
- **Input:** From **Run workflow** and later from **Append to Google Sheets** and **Search Google Maps with query** (looping behavior).
- **Outputs:**
  - Output 1 (batch items) is not used.
  - Output 2 (“No items left” path in this wiring) → **Search Google Maps with query** (this is an unusual but valid wiring pattern).
- **Key requirement:** The incoming items must contain a `query` field (string).
- **Edge cases / failures:**
  - If input items do not include `query`, downstream HTTP body expression falls back to `$('Loop over queries').item.json.query`, which can still fail if the node has no current item context.
  - Default batch sizing may cause long runs if many queries; consider explicit batch size.

---

### Block 2 — Google Maps Search + Pagination
**Overview:** For each query, calls Google Places API `places:searchText`, accumulates all pages using `nextPageToken`, and outputs a single item containing an accumulated `places[]` array.

**Nodes involved:**
- **Search Google Maps with query**
- **Check for Next Page Token**
- **Has Next Page?**

#### Node: Search Google Maps with query
- **Type / role:** HTTP Request; calls Google Places API v1 Text Search.
- **Config choices (interpreted):**
  - **POST** to `https://places.googleapis.com/v1/places:searchText`
  - JSON body includes:
    - `textQuery`: from `$json.query` (or fallback to the current batch item’s query)
    - `pageToken`: included only when present
  - Sends headers:
    - `Content-Type: application/json`
    - `X-Goog-FieldMask`: limits response fields to reduce payload:
      - `places.displayName, places.formattedAddress, places.rating, places.nationalPhoneNumber, places.regularOpeningHours, places.websiteUri, places.businessStatus, nextPageToken`
  - Auth: **HTTP Header Auth** via “genericCredentialType” (typically used to pass `X-Goog-Api-Key` or `Authorization: Bearer ...`, depending on how you configured credentials).
  - Reliability:
    - `retryOnFail: true`, `maxTries: 2`
    - `onError: continueErrorOutput`
    - `alwaysOutputData: true`
- **Inputs:** From **Loop over queries** (and also from **Has Next Page?** when continuing pagination).
- **Outputs:** Main → **Check for Next Page Token**; also wired to **Loop over queries** (used as a loop control path).
- **Edge cases / failures:**
  - Auth errors (401/403) if API key/header is wrong, Places API not enabled, or billing not configured.
  - Quota/rate limiting (429).
  - Pagination token timing: Places tokens sometimes require a short delay before reuse; this workflow does not add a wait node, so occasional “INVALID_REQUEST” style behavior may occur.
  - With `continueErrorOutput` + `alwaysOutputData`, downstream code must tolerate missing `places`/`nextPageToken`.

#### Node: Check for Next Page Token
- **Type / role:** Code; accumulates Places results across pages using `$execution.customData`.
- **Config choices:**
  - Reads response from `$input.first().json`
  - Extracts `nextPageToken` and `places` array
  - Appends places into `$execution.customData.accumulatedPlaces`
  - Returns one item containing:
    - `query`
    - `nextPageToken`
    - `pageToken` (same as nextPageToken)
    - `places` (accumulated)
    - counters: `currentPagePlaces`, `totalAccumulatedPlaces`
- **Input:** From **Search Google Maps with query**.
- **Output:** → **Has Next Page?**
- **Version-specific notes:** Uses `$execution.customData` (supported in modern n8n Code node). Behavior depends on execution mode; in parallel executions, ensure customData isolation is per execution (it is in n8n).
- **Edge cases / failures:**
  - If the HTTP node returns an error payload without `places`, `places` becomes `[]` and accumulation continues.
  - Accumulation is **never reset per query** in this code. Unless the execution is restarted or customData is manually cleared, places from multiple queries can be concatenated together. This is a critical logic risk: you typically need to reset `$execution.customData.accumulatedPlaces = []` when starting a new query.

#### Node: Has Next Page?
- **Type / role:** IF; decides whether to paginate again.
- **Condition:** `nextPageToken` is not empty.
- **Inputs:** From **Check for Next Page Token**.
- **Outputs:**
  - **True** → **Search Google Maps with query** (fetch next page)
  - **False** → **Extract Places Data** (proceed with accumulated results)
- **Edge cases:**
  - If token exists but is not yet valid, next request may return no data; consider adding a Wait (2 seconds) before calling next page.

---

### Block 3 — Business Filtering & Field Preparation
**Overview:** Splits accumulated places into individual items, keeps only OPERATIONAL listings, extracts important fields, and branches depending on website presence.

**Nodes involved:**
- **Extract Places Data**
- **Split Out Places**
- **Filter Operational Businesses**
- **Check if Website Exists**

#### Node: Extract Places Data
- **Type / role:** Set; maps `places.*` fields to flatter business fields while keeping original data.
- **Config choices:**
  - Adds fields:
    - `name` = `places.displayName.text`
    - `address` = `places.formattedAddress`
    - `rating` = `places.rating`
    - `phone` = `places.nationalPhoneNumber`
    - `openingHours` = `places.regularOpeningHours`
    - `website` = `places.websiteUri`
    - `businessStatus` = `places.businessStatus`
  - `includeOtherFields: true` (so `places` object remains too)
- **Input:** From **Has Next Page?** (false path).
- **Outputs:** 
  - → **Split Out Places**
  - → **Loop over queries** (used as workflow control/looping)
- **Edge cases:**
  - If upstream item structure is “accumulated places” (array) rather than a single `places` object, these expressions can misalign. In this workflow, the item at this stage contains `places: [...]` (array), so `places.displayName.text` is not valid until after **Split Out Places**. This suggests the Set node is placed too early or expects a different structure.

#### Node: Split Out Places
- **Type / role:** Split Out; converts `places[]` array into one item per place.
- **Config choices:**
  - `fieldToSplitOut: places`
  - `include: allOtherFields` (keeps query metadata on each item)
- **Input:** From **Extract Places Data**.
- **Output:** → **Filter Operational Businesses**
- **Edge cases:** If `places` is missing or not an array, it outputs nothing.

#### Node: Filter Operational Businesses
- **Type / role:** Filter; keeps only active listings.
- **Condition:** `places.businessStatus` equals `OPERATIONAL`.
- **Input:** From **Split Out Places**.
- **Outputs:**
  - → **Check if Website Exists**
  - → **Merge Emails with Business Data** (input 0) (passes business items forward for later merge)
- **Edge cases:** Some Places entries may not have `businessStatus`; strict validation may drop them.

#### Node: Check if Website Exists
- **Type / role:** IF; only scrape if a website is available.
- **Condition:** `places.websiteUri` is not empty.
- **Input:** From **Filter Operational Businesses**.
- **Outputs:**
  - **True** → **Fetch Website HTML**
  - **False** → **Merge All Businesses** (input 1) (so businesses without websites still flow to final merge/export path)
- **Edge cases:** Empty/invalid URLs; some websites may be blocked or redirect heavily.

---

### Block 4 — Website + Contact Page Scraping
**Overview:** Downloads homepage HTML, extracts “contact-like” links, fetches those pages, and combines all HTML into one text blob for email extraction.

**Nodes involved:**
- **Fetch Website HTML**
- **Extract Contact Links**
- **Filter Contact Links**
- **Fetch Contact Page HTML**
- **Combine Homepage and Contact HTML**

#### Node: Fetch Website HTML
- **Type / role:** HTTP Request; fetch homepage HTML.
- **Config choices:**
  - URL: `places.websiteUri`
  - `onError: continueRegularOutput`
  - Response configured with `neverError: true` (so it returns even if non-2xx)
- **Input:** From **Check if Website Exists** (true).
- **Outputs:** 
  - → **Extract Contact Links**
  - → **Combine Homepage and Contact HTML** (directly feeds homepage into the combiner)
- **Edge cases / failures:**
  - SSL/TLS errors, redirects, bot protection, large pages.
  - No custom User-Agent here (some sites block default agents).
  - If response body field naming differs (`data` vs `body`), downstream code tries multiple fallbacks.

#### Node: Extract Contact Links
- **Type / role:** Code; parses homepage HTML to find links likely leading to contact/about/legal pages, and emits them as separate items.
- **Config choices:**
  - Reads HTML from `$input.item.json.data`
  - Builds `baseUrl` from `places.websiteUri`
  - Extracts all `href="..."` occurrences
  - Filters by keywords: `contact, about, reach, info, get-in-touch, touch, connect, team, privacy, terms, legal, imprint, impressum`
  - Converts relative links to absolute
  - Outputs:
    - multiple items with `contactLink`, or one item with `contactLink: ''` if none found
- **Input:** From **Fetch Website HTML**.
- **Output:** → **Filter Contact Links**
- **Edge cases:**
  - Some sites use JS-rendered navigation; HTML won’t contain links.
  - `href` regex can capture mailto/tel/javascript links; those may slip through unless filtered later.

#### Node: Filter Contact Links
- **Type / role:** Filter; keeps non-empty contactLink and drops static assets.
- **Conditions:**
  - `contactLink` not empty
  - `contactLink` not matching `\.(png|jpg|jpeg|ico|css|svg)$`
- **Input:** From **Extract Contact Links**.
- **Output:** → **Fetch Contact Page HTML**
- **Edge cases:** Case sensitivity is strict; uppercase extensions might pass unless normalized.

#### Node: Fetch Contact Page HTML
- **Type / role:** HTTP Request; fetches contact/about/legal pages HTML.
- **Config choices:**
  - URL: `contactLink`
  - `allowUnauthorizedCerts: true`
  - `retryOnFail: true`, `maxTries: 2`
  - `neverError: true`
  - Custom `User-Agent` header (Chrome-like)
  - `onError: continueRegularOutput`
- **Input:** From **Filter Contact Links**.
- **Output:** → **Combine Homepage and Contact HTML**
- **Edge cases / failures:**
  - Some servers block repeated requests; consider rate limiting.
  - `allowUnauthorizedCerts: true` increases tolerance but reduces security.

#### Node: Combine Homepage and Contact HTML
- **Type / role:** Code; merges homepage HTML + all contact page HTML into one `combinedHtml` field while keeping business fields.
- **Config choices:**
  - Iterates over `$input.all()`, concatenates any `data/body/html`
  - Uses the first item as canonical business data
  - Removes `data/body/html/error/contactLink` fields from final output
- **Inputs:** From **Fetch Website HTML** and **Fetch Contact Page HTML** (fan-in).
- **Output:** → **Extract Emails from HTML**
- **Edge cases:**
  - Correctness depends on n8n’s item grouping; if homepage and contact pages are not in the same “merge context”, combining may mix unrelated items. Typically you’d use Merge (by key) or Wait/Combine patterns; this code node assumes the incoming items belong to the same business.

---

### Block 5 — Email Extraction & Filtering
**Overview:** Extracts emails from combined HTML, outputs one row per email (or blank), then filters out common junk/non-lead emails.

**Nodes involved:**
- **Extract Emails from HTML**
- **Filter Valid Emails**

#### Node: Extract Emails from HTML
- **Type / role:** Code; regex email extraction and normalization.
- **Config choices:**
  - HTML source fallback order: `combinedHtml`, `data`, `body`, `html`
  - Removes HTML fields from output to keep payload small
  - Regex: `/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g`
  - Dedupes emails per item and filters out emails ending with image extensions
  - Outputs:
    - If none: single item with `email: ''`
    - If multiple: multiple items with same business fields and different `email`
- **Input:** From **Combine Homepage and Contact HTML**.
- **Output:** → **Filter Valid Emails**
- **Edge cases:**
  - Obfuscated emails (“name [at] domain”) won’t match.
  - Can capture false positives in scripts or structured data.

#### Node: Filter Valid Emails
- **Type / role:** Filter; removes obvious non-contact or irrelevant addresses.
- **Condition:** `email` does **not** match regex containing:
  - `google|gstatic|ggpht|schema.org|example.com|png|jpg|gif|jpeg|wixpress|sentry|imli|noreply|no-reply|donotreply|mailer-daemon|postmaster`
- **Input:** From **Extract Emails from HTML**.
- **Output:** → **Group by Business**
- **Edge cases:**
  - This condition is “notRegex” only; it does not explicitly enforce non-empty emails, so blank `email: ''` may pass (blank usually does not match the regex, so it passes). If you want to exclude blanks, add `email notEmpty`.

---

### Block 6 — Email Quality Scoring & Best Selection
**Overview:** Groups emails per business, scores each email, sorts by score descending, keeps one, and deduplicates emails across the whole run.

**Nodes involved:**
- **Group by Business**
- **Score and Rank Emails**
- **Sort by Email Quality**
- **Keep Best Email Only**
- **Remove Duplicate Emails**

#### Node: Group by Business
- **Type / role:** Aggregate; aggregates all item data (intended to group emails per business).
- **Config choices:** `aggregateAllItemData` (aggregates all incoming items together).
- **Input:** From **Filter Valid Emails**.
- **Output:** → **Score and Rank Emails**
- **Edge cases / design risk:**
  - This configuration aggregates **everything into one group**, not “per business”. If the intent is “one group per business”, you’d typically group by `places.displayName.text` or a place ID. As-is, subsequent “keep best email only” may keep only one email for the entire dataset rather than per business (depending on how n8n outputs aggregated structures).

#### Node: Score and Rank Emails
- **Type / role:** Code; computes `emailQualityScore` (0–100).
- **Scoring logic:**
  - Starts at 100
  - -40 for generic prefixes (info/contact/admin/support/sales/etc.)
  - +20 if local part includes a dot and doesn’t include generic prefixes (proxy for personal name)
  - +30 if email domain matches website domain; -20 if mismatched
  - -25 if free provider (gmail, yahoo, hotmail, outlook, etc.)
  - clamps score to 0..100
- **Input:** From **Group by Business**.
- **Output:** → **Sort by Email Quality**
- **Edge cases:**
  - If `website` isn’t present, domain match check is skipped.
  - If input structure from Aggregate is not the expected flat item, expressions may not exist.

#### Node: Sort by Email Quality
- **Type / role:** Sort; highest score first.
- **Config choices:** Sort field `emailQualityScore` descending.
- **Input:** From **Score and Rank Emails**.
- **Output:** → **Keep Best Email Only**

#### Node: Keep Best Email Only
- **Type / role:** Limit; keep first N items (defaults to 1).
- **Config choices:** No explicit limit value shown; node default is typically 1.
- **Input:** From **Sort by Email Quality**.
- **Output:** → **Remove Duplicate Emails**
- **Edge cases:** If the previous aggregation/sort yields only one global list, this keeps only one email total.

#### Node: Remove Duplicate Emails
- **Type / role:** Remove Duplicates; deduplicates by `email`.
- **Config choices:** `compare: selectedFields`, `fieldsToCompare: email`.
- **Input:** From **Keep Best Email Only**.
- **Output:** → **Merge Emails with Business Data** (input 1)
- **Edge cases:** If blank emails are present, they may collapse into one (since all blanks are equal).

---

### Block 7 — Merge, Deduplicate Businesses, Export
**Overview:** Merges business records with extracted/scored emails, deduplicates businesses, appends to Google Sheets, and loops back to process the next query batch.

**Nodes involved:**
- **Merge Emails with Business Data**
- **Merge All Businesses**
- **Remove Duplicate Businesses**
- **Append to Google Sheets**

#### Node: Merge Emails with Business Data
- **Type / role:** Merge (Combine by Position); joins business stream with email stream item-by-item.
- **Config choices:** `mode: combine`, `combineByPosition`.
- **Inputs:**
  - Input 0: from **Filter Operational Businesses** (business items)
  - Input 1: from **Remove Duplicate Emails** (email-enriched items)
- **Output:** → **Merge All Businesses**
- **Edge cases / design risk:**
  - Combine-by-position is fragile: if counts differ (e.g., some businesses have no emails, or multiple emails), rows misalign and wrong emails may attach to wrong businesses.
  - More robust: merge by a stable key (place ID / website / name+address).

#### Node: Merge All Businesses
- **Type / role:** Merge; combines:
  - businesses with websites (and possibly emails) with
  - businesses without websites (from IF false branch)
- **Inputs:**
  - Input 0: from **Merge Emails with Business Data**
  - Input 1: from **Check if Website Exists** (false path)
- **Output:** → **Remove Duplicate Businesses**
- **Edge cases:** If both inputs contain overlapping structures, you may get duplicates without strong keys.

#### Node: Remove Duplicate Businesses
- **Type / role:** Remove Duplicates; deduplicates by business name.
- **Config choices:** compare selected field `places.displayName.text`.
- **Input:** From **Merge All Businesses**.
- **Output:** → **Append to Google Sheets**
- **Edge cases / data quality:**
  - Business name alone is not unique (chains/duplicates across cities). Better key: Google Place ID (not requested in FieldMask) or name+address.

#### Node: Append to Google Sheets
- **Type / role:** Google Sheets; append rows to a sheet.
- **Config choices (interpreted):**
  - Operation: **Append**
  - Document: “Solar leads” (Spreadsheet ID `1pBTh42nVAp3X1OTgTsdWVAc-YUosIPI4vGRilqWJMYY`)
  - Sheet tab: “B2B Lead Scraper” (gid `509043379`)
  - Column mapping includes:
    - `name` = `places.displayName.text`
    - `email` = `email`
    - `phone` = `places.nationalPhoneNumber`
    - `places` = `places.displayName.languageCode` (this looks unintended; likely meant to store query/city/category)
    - `address` = `places.formattedAddress`
    - `website` = `places.websiteUri`
    - `businessStatus` = `places.businessStatus`
  - Note: `rating` exists in schema but marked removed; it is not mapped in `value` currently.
- **Input:** From **Remove Duplicate Businesses**.
- **Output:** → **Loop over queries** (continues loop)
- **Credentials:** Google Sheets OAuth2 (must be connected in n8n).
- **Edge cases / failures:**
  - Permission errors if the OAuth account lacks access to the spreadsheet.
  - Column schema mismatch if sheet headers differ.
  - Quotas / rate limits on Sheets API for large batches.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run workflow | Manual Trigger | Entry point | — | Loop over queries | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Loop over queries | Split In Batches | Iterates over query items | Run workflow; Append to Google Sheets; Search Google Maps with query; Extract Places Data | Search Google Maps with query | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Search Google Maps with query | HTTP Request | Places API text search call | Loop over queries; Has Next Page? | Check for Next Page Token; Loop over queries | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Check for Next Page Token | Code | Extract/accumulate places + token | Search Google Maps with query | Has Next Page? | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Has Next Page? | IF | Pagination decision | Check for Next Page Token | Search Google Maps with query; Extract Places Data | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Extract Places Data | Set | Map place fields to lead fields | Has Next Page? | Split Out Places; Loop over queries | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Split Out Places | Split Out | One item per place | Extract Places Data | Filter Operational Businesses | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Filter Operational Businesses | Filter | Keep OPERATIONAL listings | Split Out Places | Check if Website Exists; Merge Emails with Business Data | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Check if Website Exists | IF | Branch: scrape vs no website | Filter Operational Businesses | Fetch Website HTML; Merge All Businesses | ## Stage 2: Website Scraping<br><br>Visits business websites and contact pages to extract all available email addresses. Handles both homepage and dedicated contact page scraping. |
| Fetch Website HTML | HTTP Request | Download homepage HTML | Check if Website Exists | Extract Contact Links; Combine Homepage and Contact HTML | ## Stage 2: Website Scraping<br><br>Visits business websites and contact pages to extract all available email addresses. Handles both homepage and dedicated contact page scraping. |
| Extract Contact Links | Code | Find contact/about/legal links | Fetch Website HTML | Filter Contact Links | ## Stage 2: Website Scraping<br><br>Visits business websites and contact pages to extract all available email addresses. Handles both homepage and dedicated contact page scraping. |
| Filter Contact Links | Filter | Keep usable contact links | Extract Contact Links | Fetch Contact Page HTML | ## Stage 2: Website Scraping<br><br>Visits business websites and contact pages to extract all available email addresses. Handles both homepage and dedicated contact page scraping. |
| Fetch Contact Page HTML | HTTP Request | Download contact page HTML | Filter Contact Links | Combine Homepage and Contact HTML | ## Stage 2: Website Scraping<br><br>Visits business websites and contact pages to extract all available email addresses. Handles both homepage and dedicated contact page scraping. |
| Combine Homepage and Contact HTML | Code | Combine HTML sources per business | Fetch Website HTML; Fetch Contact Page HTML | Extract Emails from HTML | ## Stage 2: Website Scraping<br><br>Visits business websites and contact pages to extract all available email addresses. Handles both homepage and dedicated contact page scraping. |
| Extract Emails from HTML | Code | Regex extract emails | Combine Homepage and Contact HTML | Filter Valid Emails | ## Stage 3: Email Quality Control<br><br>Scores emails by quality (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers). Keeps only the best email per business. |
| Filter Valid Emails | Filter | Remove junk/non-lead emails | Extract Emails from HTML | Group by Business | ## Stage 3: Email Quality Control<br><br>Scores emails by quality (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers). Keeps only the best email per business. |
| Group by Business | Aggregate | Aggregate email items (intended per business) | Filter Valid Emails | Score and Rank Emails | ## Stage 3: Email Quality Control<br><br>Scores emails by quality (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers). Keeps only the best email per business. |
| Score and Rank Emails | Code | Compute `emailQualityScore` | Group by Business | Sort by Email Quality | ## Stage 3: Email Quality Control<br><br>Scores emails by quality (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers). Keeps only the best email per business. |
| Sort by Email Quality | Sort | Best score first | Score and Rank Emails | Keep Best Email Only | ## Stage 3: Email Quality Control<br><br>Scores emails by quality (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers). Keeps only the best email per business. |
| Keep Best Email Only | Limit | Keep top 1 email | Sort by Email Quality | Remove Duplicate Emails | ## Stage 3: Email Quality Control<br><br>Scores emails by quality (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers). Keeps only the best email per business. |
| Remove Duplicate Emails | Remove Duplicates | Deduplicate by email | Keep Best Email Only | Merge Emails with Business Data | ## Stage 3: Email Quality Control<br><br>Scores emails by quality (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers). Keeps only the best email per business. |
| Merge Emails with Business Data | Merge | Join business + email streams | Filter Operational Businesses; Remove Duplicate Emails | Merge All Businesses | ## Stage 4: Final Output & Deduplication<br><br>Merges email data with business info, removes duplicate businesses, and exports clean leads to Google Sheets. |
| Merge All Businesses | Merge | Combine with no-website businesses | Merge Emails with Business Data; Check if Website Exists | Remove Duplicate Businesses | ## Stage 4: Final Output & Deduplication<br><br>Merges email data with business info, removes duplicate businesses, and exports clean leads to Google Sheets. |
| Remove Duplicate Businesses | Remove Duplicates | Deduplicate businesses by name | Merge All Businesses | Append to Google Sheets | ## Stage 4: Final Output & Deduplication<br><br>Merges email data with business info, removes duplicate businesses, and exports clean leads to Google Sheets. |
| Append to Google Sheets | Google Sheets | Export leads | Remove Duplicate Businesses | Loop over queries | ## Stage 4: Final Output & Deduplication<br><br>Merges email data with business info, removes duplicate businesses, and exports clean leads to Google Sheets. |
| 📋 Overview: B2B Lead Generation Scraper | Sticky Note | Documentation / context | — | — | ## B2B Lead Generation Scraper<br><br>This workflow automates lead discovery and qualification from Google Maps. It searches for businesses, scrapes their websites for contact emails, scores email quality, and exports verified leads to Google Sheets.<br><br>### How it works<br>1. Input search queries (e.g., "Dentist New York", "Plumber London")<br>2. Searches Google Maps for all matching businesses<br>3. Filters for operational businesses only<br>4. Visits business websites to find contact emails<br>5. Scores emails by quality (domain match, personal names, generic prefixes)<br>6. Keeps the best email per business<br>7. Exports deduplicated leads to Google Sheets<br><br>### Setup steps<br>1. Get Google Places API key from Google Cloud Console<br>2. Enable Places API (New) in your project<br>3. Add credentials to "Search Google Maps with query" node<br>4. Connect Google Sheets credentials for lead export<br>5. Prepare search queries in JSON format (see Query Prompt sticky)<br>6. Adjust email scoring thresholds if needed |
| Stage 1: Google Maps Search | Sticky Note | Documentation / stage label | — | — | ## Stage 1: Google Maps Search<br><br>Loops through search queries, fetches all matching businesses from Google Maps, handles pagination automatically, and filters for operational businesses only. |
| Stage 2: Website Scraping | Sticky Note | Documentation / stage label | — | — | ## Stage 2: Website Scraping<br><br>Visits business websites and contact pages to extract all available email addresses. Handles both homepage and dedicated contact page scraping. |
| Stage 3: Email Quality Control | Sticky Note | Documentation / stage label | — | — | ## Stage 3: Email Quality Control<br><br>Scores emails by quality (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers). Keeps only the best email per business. |
| Stage 4: Final Output | Sticky Note | Documentation / stage label | — | — | ## Stage 4: Final Output & Deduplication<br><br>Merges email data with business info, removes duplicate businesses, and exports clean leads to Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Run workflow”**
   - Node: **Manual Trigger**
   - No configuration.

2. **Create “Loop over queries”**
   - Node: **Split In Batches**
   - Keep defaults (optionally set Batch Size).
   - Connect: **Run workflow → Loop over queries**.

3. **Prepare query input items**
   - When executing manually, ensure incoming items look like:
     - `{ "query": "Dentist New York" }`
     - `{ "query": "Plumber London" }`
   - In practice, you’d usually add a Set node before the batch node to generate these items.

4. **Create “Search Google Maps with query”**
   - Node: **HTTP Request**
   - Method: **POST**
   - URL: `https://places.googleapis.com/v1/places:searchText`
   - Authentication: **Header Auth** (create credential that sends your API key header; commonly `X-Goog-Api-Key: <key>`).
   - Headers:
     - `Content-Type: application/json`
     - `X-Goog-FieldMask: places.displayName,places.formattedAddress,places.rating,places.nationalPhoneNumber,places.regularOpeningHours,places.websiteUri,places.businessStatus,nextPageToken`
   - Body: JSON with expression:
     - `textQuery` from current item’s `query`
     - include `pageToken` only when present
   - Enable retry (max tries 2), and set error handling to continue.
   - Connect: **Loop over queries → Search Google Maps with query**.

5. **Create “Check for Next Page Token”**
   - Node: **Code**
   - Paste logic that:
     - reads response, extracts `nextPageToken` and `places`
     - accumulates into `$execution.customData.accumulatedPlaces`
     - returns `{ query, nextPageToken, pageToken, places }`
   - Connect: **Search Google Maps with query → Check for Next Page Token**.

6. **Create “Has Next Page?”**
   - Node: **IF**
   - Condition: `{{$json.nextPageToken}}` **not empty**
   - Connect:
     - **Check for Next Page Token → Has Next Page?**
     - **True → Search Google Maps with query** (pagination loop)
     - **False → Extract Places Data**

7. **Create “Extract Places Data”**
   - Node: **Set**
   - Add mapped fields (name/address/rating/phone/openingHours/website/businessStatus) from `places.*`
   - Keep “Include Other Fields” enabled.
   - Connect: **Has Next Page? (false) → Extract Places Data**

8. **Create “Split Out Places”**
   - Node: **Split Out**
   - Field to split: `places`
   - Include: “All other fields”
   - Connect: **Extract Places Data → Split Out Places**

9. **Create “Filter Operational Businesses”**
   - Node: **Filter**
   - Condition: `{{$json.places.businessStatus}}` equals `OPERATIONAL`
   - Connect: **Split Out Places → Filter Operational Businesses**

10. **Create “Check if Website Exists”**
    - Node: **IF**
    - Condition: `{{$json.places.websiteUri}}` not empty
    - Connect: **Filter Operational Businesses → Check if Website Exists**

11. **Create “Fetch Website HTML”**
    - Node: **HTTP Request**
    - URL: `{{$json.places.websiteUri}}`
    - Configure to not fail on non-2xx (never error) and continue on error.
    - Connect: **Check if Website Exists (true) → Fetch Website HTML**

12. **Create “Extract Contact Links”**
    - Node: **Code (run once for each item)**
    - Parse `$json.data` for href links; filter by contact keywords; output items with `contactLink`.
    - Connect: **Fetch Website HTML → Extract Contact Links**

13. **Create “Filter Contact Links”**
    - Node: **Filter**
    - `contactLink` not empty
    - `contactLink` notRegex for static extensions
    - Connect: **Extract Contact Links → Filter Contact Links**

14. **Create “Fetch Contact Page HTML”**
    - Node: **HTTP Request**
    - URL: `{{$json.contactLink}}`
    - Add User-Agent header
    - allowUnauthorizedCerts: true
    - retryOnFail: true (max tries 2)
    - neverError true / continue on error
    - Connect: **Filter Contact Links → Fetch Contact Page HTML**

15. **Create “Combine Homepage and Contact HTML”**
    - Node: **Code**
    - Combine all incoming HTML strings into `combinedHtml` while keeping business fields.
    - Connect:
      - **Fetch Website HTML → Combine Homepage and Contact HTML**
      - **Fetch Contact Page HTML → Combine Homepage and Contact HTML**

16. **Create “Extract Emails from HTML”**
    - Node: **Code**
    - Use regex to extract emails from `combinedHtml` (fallbacks permitted)
    - Output multiple items for multiple emails; otherwise `email: ''`
    - Connect: **Combine Homepage and Contact HTML → Extract Emails from HTML**

17. **Create “Filter Valid Emails”**
    - Node: **Filter**
    - Condition: `email` notRegex with junk patterns (google/gstatic/noreply/etc.)
    - (Recommended) add `email notEmpty`
    - Connect: **Extract Emails from HTML → Filter Valid Emails**

18. **Create “Group by Business”**
    - Node: **Aggregate**
    - Configure grouping behavior (recommended: group by a unique key like Place ID; if not available, use name+address).
    - Connect: **Filter Valid Emails → Group by Business**

19. **Create “Score and Rank Emails”**
    - Node: **Code**
    - Compute `emailQualityScore` using website-domain match, generic prefixes, free providers, etc.
    - Connect: **Group by Business → Score and Rank Emails**

20. **Create “Sort by Email Quality”**
    - Node: **Sort**
    - Sort by `emailQualityScore` descending
    - Connect: **Score and Rank Emails → Sort by Email Quality**

21. **Create “Keep Best Email Only”**
    - Node: **Limit**
    - Set limit to 1 (or confirm default)
    - Connect: **Sort by Email Quality → Keep Best Email Only**

22. **Create “Remove Duplicate Emails”**
    - Node: **Remove Duplicates**
    - Compare field: `email`
    - Connect: **Keep Best Email Only → Remove Duplicate Emails**

23. **Create “Merge Emails with Business Data”**
    - Node: **Merge**
    - Mode: **Combine by Position** (as in the workflow; recommended to merge by key instead)
    - Connect:
      - **Filter Operational Businesses → Merge Emails with Business Data** (input 0)
      - **Remove Duplicate Emails → Merge Emails with Business Data** (input 1)

24. **Create “Merge All Businesses”**
    - Node: **Merge**
    - Connect:
      - **Merge Emails with Business Data → Merge All Businesses** (input 0)
      - **Check if Website Exists (false) → Merge All Businesses** (input 1)

25. **Create “Remove Duplicate Businesses”**
    - Node: **Remove Duplicates**
    - Compare field: `places.displayName.text`
    - Connect: **Merge All Businesses → Remove Duplicate Businesses**

26. **Create “Append to Google Sheets”**
    - Node: **Google Sheets**
    - Operation: **Append**
    - Credentials: Google Sheets OAuth2 (account must have edit access)
    - Select Spreadsheet + Sheet tab
    - Map columns (name/email/phone/address/website/businessStatus, etc.)
    - Connect: **Remove Duplicate Businesses → Append to Google Sheets**

27. **Close the loop**
    - Connect: **Append to Google Sheets → Loop over queries** (to continue with the next query).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Enable “Places API (New)” and configure billing; create a Places API key | Required for Google Places `places:searchText` endpoint |
| Use `X-Goog-FieldMask` to reduce payload and cost | Implemented in “Search Google Maps with query” |
| Email quality scoring rules | Embedded in “Score and Rank Emails” (+30 domain match, +20 personal names, -40 generic prefixes, -25 free providers) |
| Export destination | Google Sheets document “Solar leads”, sheet “B2B Lead Scraper” (as configured in the node) |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.