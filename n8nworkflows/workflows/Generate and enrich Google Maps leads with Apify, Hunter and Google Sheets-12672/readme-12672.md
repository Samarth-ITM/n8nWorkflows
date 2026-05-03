Generate and enrich Google Maps leads with Apify, Hunter and Google Sheets

https://n8nworkflows.xyz/workflows/generate-and-enrich-google-maps-leads-with-apify--hunter-and-google-sheets-12672


# Generate and enrich Google Maps leads with Apify, Hunter and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate and enrich Google Maps leads with Apify, Hunter and Google Sheets

**Purpose:**  
A scheduled lead-generation pipeline that:
1) Scrapes Google Maps business leads via **Apify** (HTTP request),  
2) Cleans/normalizes and quality-scores the results,  
3) Deduplicates against existing leads stored in **Google Sheets**,  
4) Enriches leads with emails using **Hunter.io domain search**,  
5) Generates an HTML summary report and emails it via **Gmail**,  
6) Saves enriched leads back into **Google Sheets** (append or update).

**Primary use cases:**
- Daily lead collection for a niche and geography (configured: “digital marketing agency” in “Mumbai, India”).
- Automated enrichment and prioritization based on email confidence.
- Ongoing CRM-like storage in Google Sheets with deduplication.

### 1.1 Scheduled Run & Scraping (Apify)
Runs daily at 09:00, calls Apify actor endpoint to scrape Google Maps results.

### 1.2 Data Cleaning, Quality Filtering & Deduplication (Sheets)
Normalizes the scraped output, computes a data quality score, filters low-quality leads, loads existing sheet leads, and removes duplicates.

### 1.3 Batch Enrichment (Hunter)
Processes new leads in batches of 5; for each lead, calls Hunter domain-search using the lead’s website domain and merges enrichment results.

### 1.4 Classification, Reporting & Persistence
Splits leads into High vs Medium/Low confidence groups, generates a rich HTML email report, sends it, prepares sheet rows, and appends/updates Google Sheets.

### 1.5 Error Handling & Notifications
Multiple nodes use “continue on error” paths routed through an error formatter and an email notification sender.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Trigger & Google Maps Scrape
**Overview:** Starts the workflow daily and fetches Google Maps leads using Apify’s API.  
**Nodes involved:** `Schedule Trigger1`, `Google Maps Scraper`

#### Node: Schedule Trigger1
- **Type / Role:** Schedule Trigger; entry point.
- **Configuration (interpreted):** Runs every day at **09:00** (triggerAtHour = 9).
- **Connections:** Outputs to `Google Maps Scraper`.
- **Edge cases / failures:**
  - Timezone depends on n8n instance settings; may not match user locale.
  - If the instance is down at 09:00, execution may be missed depending on n8n scheduling behavior/version.

#### Node: Google Maps Scraper
- **Type / Role:** HTTP Request; calls Apify Actor run endpoint (sync + dataset items).
- **Configuration:**
  - **POST** to: `https://api.apify.com/v2/acts/account-id/run-sync-get-dataset-items`
  - Timeout: **300000 ms** (5 minutes)
  - JSON body includes:
    - `searchStringsArray`: `[ { searchString: "digital marketing agency", location: "Mumbai, India" } ]`
    - `maxCrawledPlacesPerSearch`: `20`
  - Headers:
    - `Accept: application/json`
    - `Authorization: Bearer YOUR_TOKEN_HERE` (must be replaced with a real Apify token)
  - **onError:** `continueErrorOutput` (routes errors to a secondary output)
- **Connections:**
  - **Success output:** to `Format & Validate Data`
  - **Error output:** to `Format Error Data1`
- **Edge cases / failures:**
  - 401/403 if token is invalid/expired.
  - 404 if actor endpoint/account-id is wrong.
  - 429 rate limits or Apify capacity issues.
  - Response format variations (handled partly downstream).

---

### Block 2 — Normalize, Score, Filter & Deduplicate
**Overview:** Validates scraper response, normalizes fields, computes a quality score, filters weak leads, loads existing leads from Sheets, and removes duplicates.  
**Nodes involved:** `Format & Validate Data`, `Check Existing Leads1`, `Deduplicate Leads1`, `Has New Leads?1`, `No New Leads Notification1`

#### Node: Format & Validate Data
- **Type / Role:** Code node; normalization + quality scoring + filtering.
- **Configuration choices:**
  - Validates that input exists; throws if empty.
  - Attempts to detect businesses array:
    - If `rawJson[0].items` is an array, uses that.
    - Else if the raw payload itself is an array, uses it.
    - Else throws “Invalid response format”.
  - Maps fields from possible Apify shapes:
    - `business_name`: `title || name`
    - `address`: `address || formattedAddress`
    - `phone`: `phone || phoneNumber`
    - `website`: `website || websiteUrl`
    - `google_maps_url`: `url || placeUrl`
    - `category`: `categories.join(", ")` or `category`
    - `rating`: `rating || totalScore`
    - `reviews_count`: `reviewsCount || reviews`
    - `latitude`: `latitude || lat`, `longitude`: `longitude || lng`
  - Computes `data_quality_score` (0–100):
    - +25 name, +20 phone, +30 website, +15 address, +10 rating>=3.5
  - Filters final leads:
    - must have `business_name`
    - must have `google_maps_url`
    - `data_quality_score >= 40`
- **Key variables:** `finalLeads`, `data_quality_score`
- **Connections:**
  - Main output to `Check Existing Leads1`
  - Error output to `Format Error Data1` (because node has `onError: continueErrorOutput`)
- **Edge cases / failures:**
  - Throws errors for empty/no businesses after filtering → will go to error branch.
  - Some websites may be empty; later Hunter lookup assumes `website` exists and is a string (risk addressed in Block 3).

#### Node: Check Existing Leads1
- **Type / Role:** Google Sheets node; reads existing leads for deduplication.
- **Configuration:**
  - Targets a Google Sheets **documentId** (`google-sheets-document-id`) and sheet `gid=0` (via list selector).
  - Operation isn’t explicitly shown in the snippet, but based on usage it returns rows (typical “Read” behavior).
  - **onError:** `continueRegularOutput` (so failures won’t stop; it will continue with whatever output is produced).
- **Connections:** Outputs to `Deduplicate Leads1`.
- **Edge cases / failures:**
  - Auth/permission issues (OAuth scope, shared sheet access).
  - If it fails and returns no items, deduplication will treat as empty and may create duplicates.

#### Node: Deduplicate Leads1
- **Type / Role:** Code node; removes leads already present in Sheets.
- **Logic:**
  - New leads come from `$items("Format & Validate Data")`.
  - Existing leads from `$items("Check Existing Leads1") || []`.
  - Builds a Set keyed by: `business|website` (lowercased). Reads sheet columns as `business/Business` and `website/Website`.
  - Filters out any new lead whose `business_name|website` exists in that Set.
  - If no unique leads remain, returns a single item:
    - `{ message: "No new unique leads found", total_scraped, duplicates_filtered }`
- **Connections:** to `Has New Leads?1`.
- **Edge cases / failures:**
  - If the sheet uses different column names than expected, Set may be incomplete → duplicates slip in.
  - If `website` differs by protocol/trailing slash, duplicates may not match (no canonicalization beyond lowercasing).

#### Node: Has New Leads?1
- **Type / Role:** IF node; determines whether pipeline continues.
- **Condition:** `$json.message != "No new unique leads found"`
  - If the item is a real lead, `message` is undefined, so condition evaluates as “not equals” and passes.
- **Connections:**
  - **True:** `Batch for AI Processing1`
  - **False:** `No New Leads Notification1`
- **Edge cases:**
  - If a lead item accidentally contains `message: "No new unique leads found"`, it would be misrouted (unlikely).

#### Node: No New Leads Notification1
- **Type / Role:** Gmail node; sends an informational email when nothing new is found.
- **Configuration:**
  - To: `your-email-id`
  - Subject includes date formatting: `{{ $now.format('MMM DD, YYYY') }}`
  - Message includes `total_scraped`, `duplicates_filtered`, query/location text.
- **Edge cases / failures:**
  - Gmail OAuth not configured or revoked.
  - If dedupe output structure changes, template variables may be undefined.

---

### Block 3 — Batch Enrichment with Hunter.io
**Overview:** Batches unique leads and calls Hunter domain-search for each lead to find emails; merges/validates results with the original lead data.  
**Nodes involved:** `Batch for AI Processing1`, `HTTP Request1`, `Merge & Validate Results`

#### Node: Batch for AI Processing1
- **Type / Role:** Split in Batches; throttles API calls.
- **Configuration:** `batchSize: 5`
- **Connections (important behavior):**
  - In n8n, SplitInBatches has two outputs:
    - Output 0: batch items
    - Output 1: “No items left” (or continuation depending on version)
  - Here, **output 1** is connected to `HTTP Request1`, while **output 0 is not connected**.
- **Implication / likely issue:**
  - This wiring is unusual: normally you connect **output 0** to the processing node (Hunter request) and then loop back.
  - As configured, Hunter requests may only run when the node indicates the “done/second output” path, which can lead to **no enrichment** or incorrect looping depending on n8n version.
- **Connections:**
  - Output 1 → `HTTP Request1`
  - Later loop: `Wait1` → `Batch for AI Processing1` (see Block 5)
- **Edge cases:**
  - Potential infinite loop or zero processing if connected incorrectly.
  - If it does process, batch size limits concurrency but not necessarily API rate limit bursts without additional waits.

#### Node: HTTP Request1 (Hunter)
- **Type / Role:** HTTP Request; calls Hunter domain-search endpoint.
- **Configuration:**
  - GET/Request to: `https://api.hunter.io/v2/domain-search` (method not shown; node config shows query usage; n8n defaults to GET if not specified, but here it’s not explicit—assume GET).
  - Query parameters:
    - `domain`: `{{ $json.website.replace(/^https?:\/\//,'').replace(/\/.*$/,'') }}`
    - `limit`: `5`
  - Header: `X-Api-Key: your-api-key`
  - **onError:** `continueErrorOutput` (error output routed)
- **Connections:**
  - Success → `Merge & Validate Results`
  - Error → `Format Error Data1`
- **Edge cases / failures:**
  - If `$json.website` is empty/null, `.replace()` will throw (expression failure) and be treated as node error.
  - Hunter 401 (bad key), 429 (rate), 402/403 (plan limits), or domain-search returning empty results.
  - Hunter may return `data.emails` with various structures; downstream expects array.

#### Node: Merge & Validate Results
- **Type / Role:** Code node; merges Hunter response with original lead; validates email format; computes confidence label.
- **Logic highlights:**
  - `originalItems = $items("Format & Validate Data")`
  - `hunterItems = $input.all()`
  - For each hunter item (by index), picks `originalItems[index]`:
    - **Assumes index alignment** between Hunter results and original leads.
  - Default email data: null email, Low confidence, `hunter_not_found`.
  - If Hunter returned `data.emails`:
    - Sort by `confidence` descending
    - Confidence mapping:
      - >=85 → High
      - >=65 → Medium
      - else Low
    - Validates email with regex; invalid becomes null/Low.
  - Collects parse/API errors into `errors[]` and attaches `_errors` to the **first merged item**.
  - Outputs a normalized schema with fields used later:
    - business_name, address, phone, website, google_maps_url, category, rating, reviews_count, latitude, longitude, data_quality_score
    - email, email_confidence, email_source, email_reasoning, alternative_emails
    - processed_date, workflow_version
- **Connections:** Output → `High Confidence Leads?1`
- **Edge cases / failures:**
  - Misalignment risk: if batching changes order/quantity, index-based merge may attach wrong email to wrong business.
  - If Hunter returns fewer items than inputs (e.g., errors skipped), indexing breaks.
  - Attaching `_errors` only to first item means later nodes must read from first item to see all errors (Build Email Report does this).

---

### Block 4 — Confidence Split & Report Generation
**Overview:** Splits leads by email confidence, generates a detailed HTML report, and sends it via Gmail.  
**Nodes involved:** `High Confidence Leads?1`, `Build Email Report1`, `Send Email Report1`

#### Node: High Confidence Leads?1
- **Type / Role:** IF node; classifies leads for reporting/prioritization.
- **Conditions (AND):**
  - `email_confidence == "High"`
  - `email != "null"` (string comparison)
- **Connections:**
  - True branch (output 0) → `Build Email Report1` and `Prepare Sheet Data1`
  - False branch (output 1) → `Prepare Sheet Data1` and `Build Email Report1`
- **Important behavior:**
  - Both branches go to both downstream nodes, so **all leads** proceed to saving/reporting; the IF is mainly used for counting/grouping inside the report logic via `$items("High Confidence Leads?1", 0/1)`.
- **Edge cases:**
  - In merged data, `email` is set to `null` (actual null), not `"null"` string. The IF compares string `"null"`, so:
    - `null != "null"` is true in many coercion contexts, but n8n’s strict typing can behave unexpectedly. With `typeValidation: "strict"`, this may cause condition evaluation issues depending on n8n version.
  - Safer would be checking `isNotEmpty` or `notEquals` with empty string and also handling null.

#### Node: Build Email Report1
- **Type / Role:** Code node; constructs analytics + HTML email body.
- **Inputs used:**
  - `allLeads = $items("Merge & Validate Results")`
  - `highConfLeads = $items("High Confidence Leads?1", 0)`
  - `medLowLeads = $items("High Confidence Leads?1", 1)`
  - Reads `_errors` from `allLeads[0]?.json._errors`
- **Outputs:**
  - Summary metrics (total, withEmail, success_rate, breakdown counts, avg_quality_score, website_coverage)
  - `email_html` (large HTML string with tables)
- **Notable embedded constants:**
  - The report shows query/location hardcoded: “digital marketing agency in Mumbai, India”
- **Connections:** → `Send Email Report1`
- **Edge cases / failures:**
  - If `Merge & Validate Results` returns 0 items, referencing `allLeads[0]` could be undefined (guarded partially).
  - HTML includes a “🎯” character in `<h1>`; some clients handle fine, but keep consistent encoding.

#### Node: Send Email Report1
- **Type / Role:** Gmail node; sends the HTML report.
- **Configuration:**
  - To: `your-email`
  - Subject: `🎯 Lead Report - {{ total_leads }} Leads ({{ high_confidence }} High Priority) - {{ $now.format('MMM DD, YYYY') }}`
  - Message body: `{{ $json.email_html }}`
- **Connections:** → `Merge1`
- **Edge cases:**
  - Gmail may sanitize some HTML/CSS; tables usually ok.
  - Ensure Gmail node is set to send HTML (n8n Gmail “message” typically supports HTML; if not, use “HTML” option if available in your node version).

---

### Block 5 — Save to Google Sheets & Control Flow Loop
**Overview:** Converts final enriched lead objects to sheet columns and writes them (append/update). Then merges with email path and loops via Wait → SplitInBatches to process remaining batches.  
**Nodes involved:** `Prepare Sheet Data1`, `Save to Google Sheets`, `Merge1`, `Wait1`

#### Node: Prepare Sheet Data1
- **Type / Role:** Code node; maps enriched lead fields to a flat row structure for Sheets.
- **Mapping:**
  - Core: business, email, phone, website, location, industry
  - Enrichment metadata: email_confidence, email_source, alternative_emails, data_quality_score
  - Metrics: rating, reviews_count
  - Geo: latitude, longitude
  - Tracking: source="Google Maps", google_maps_url, scraped_date (YYYY-MM-DD), processed_timestamp, workflow_version="2.0"
- **Connections:** → `Save to Google Sheets`
- **Edge cases:**
  - If downstream sheet expects exact column names, ensure they match.

#### Node: Save to Google Sheets
- **Type / Role:** Google Sheets node; append or update rows.
- **Configuration:**
  - Operation: **appendOrUpdate**
  - Matching column: **Business**
  - Mapping mode: “define below” with explicit schema and expressions.
  - Sheet: specified by internal gid (value: `2133476898`)
  - Document: `google-sheets-document-id`
  - **onError:** `continueErrorOutput`
- **Connections:**
  - Main output → `Merge1` (index 1)
  - Error output → `Format Error Data1`
- **Important behaviors:**
  - Matching by Business alone can overwrite rows when different companies share identical names.
  - Better matching would include website/domain too.
- **Edge cases / failures:**
  - Schema mismatch if sheet headers differ (e.g., “review_counts” vs “reviews_count”; here it uses `review_counts` in schema but sources `$json.reviews_count`).
  - Permissions / API quota.

#### Node: Merge1
- **Type / Role:** Merge node; joins two branches (email report path and sheet save path).
- **Inputs:**
  - Input 0 from `Send Email Report1`
  - Input 1 from `Save to Google Sheets`
- **Output:** → `Wait1`
- **Edge cases:**
  - Merge mode not specified; default behavior depends on node version (often “Combine”/“Append” behavior). If it waits for both inputs, a failure in either branch may stall unless errors are continued.

#### Node: Wait1
- **Type / Role:** Wait node; used here as a control node to loop back into batching.
- **Configuration:** empty (default wait). It has a webhookId, implying it can resume via webhook in some modes; but no explicit wait-until time is set in parameters shown.
- **Connections:** → `Batch for AI Processing1`
- **Edge cases:**
  - If configured as “Wait for webhook” by default in your n8n version, the workflow may pause indefinitely.
  - If intended as a short delay between batches, a fixed duration should be configured explicitly.

---

### Block 6 — Error Formatting & Error Email
**Overview:** Collects error outputs from nodes configured to continue on error and sends a formatted error email.  
**Nodes involved:** `Format Error Data1`, `Error Notification1`

#### Node: Format Error Data1
- **Type / Role:** Code node; normalizes error object into a single report item.
- **Logic:**
  - Reads all input error items.
  - Extracts `item.json.error` fields: `name`, `message`, `stack`.
  - Uses `item.json.node` as `failed_node` if present.
  - Returns only the first error as `{ json: errors[0] }`.
- **Connections:** → `Error Notification1`
- **Edge cases:**
  - Error payload shape differs between nodes; `item.json.error` may not exist, resulting in “Unknown Error”.
  - Only first error is emailed; others are dropped.

#### Node: Error Notification1
- **Type / Role:** Gmail node; sends an error alert email.
- **Configuration:** Sends error_type, error_message, failed_node, timestamp.
- **Edge cases:** Gmail credential issues; missing variables if formatter didn’t find fields.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger1 | Schedule Trigger | Daily entry point | — | Google Maps Scraper | ## Step 1 – Scrape & Deduplicate<br>Runs on schedule, scrapes Google Maps businesses, cleans and scores data, and removes duplicates already stored in Google Sheets. |
| Google Maps Scraper | HTTP Request | Scrape Google Maps leads via Apify | Schedule Trigger1 | Format & Validate Data; Format Error Data1 | ## Step 1 – Scrape & Deduplicate<br>Runs on schedule, scrapes Google Maps businesses, cleans and scores data, and removes duplicates already stored in Google Sheets. |
| Format & Validate Data | Code | Normalize, quality-score, filter leads | Google Maps Scraper | Check Existing Leads1; Format Error Data1 | ## Step 1 – Scrape & Deduplicate<br>Runs on schedule, scrapes Google Maps businesses, cleans and scores data, and removes duplicates already stored in Google Sheets. |
| Check Existing Leads1 | Google Sheets | Load existing leads for dedupe | Format & Validate Data | Deduplicate Leads1 | ## Step 1 – Scrape & Deduplicate<br>Runs on schedule, scrapes Google Maps businesses, cleans and scores data, and removes duplicates already stored in Google Sheets. |
| Deduplicate Leads1 | Code | Remove duplicates vs Sheets | Check Existing Leads1 | Has New Leads?1 | ## Step 1 – Scrape & Deduplicate<br>Runs on schedule, scrapes Google Maps businesses, cleans and scores data, and removes duplicates already stored in Google Sheets. |
| Has New Leads?1 | IF | Route based on presence of unique leads | Deduplicate Leads1 | Batch for AI Processing1; No New Leads Notification1 | ## Step 1 – Scrape & Deduplicate<br>Runs on schedule, scrapes Google Maps businesses, cleans and scores data, and removes duplicates already stored in Google Sheets. |
| No New Leads Notification1 | Gmail | Informational email when no new leads | Has New Leads?1 (false) | — | ## Step 1 – Scrape & Deduplicate<br>Runs on schedule, scrapes Google Maps businesses, cleans and scores data, and removes duplicates already stored in Google Sheets. |
| Batch for AI Processing1 | Split in Batches | Batch leads for enrichment | Has New Leads?1 (true); Wait1 | HTTP Request1 (via output 1) | ## Step 2 – Enrich Emails<br>New leads are processed in batches and enriched using Hunter domain search to find and verify business emails. |
| HTTP Request1 | HTTP Request | Hunter domain-search enrichment | Batch for AI Processing1 | Merge & Validate Results; Format Error Data1 | ## Step 2 – Enrich Emails<br>New leads are processed in batches and enriched using Hunter domain search to find and verify business emails. |
| Merge & Validate Results | Code | Merge Hunter results with original leads | HTTP Request1 | High Confidence Leads?1 | ## Step 2 – Enrich Emails<br>New leads are processed in batches and enriched using Hunter domain search to find and verify business emails. |
| High Confidence Leads?1 | IF | Split by email confidence for reporting | Merge & Validate Results | Build Email Report1; Prepare Sheet Data1 | ## Step 3 – Report & Store<br>Leads are classified by confidence, emailed as a daily report, and saved to Google Sheets for long-term tracking. |
| Build Email Report1 | Code | Build HTML report + metrics | High Confidence Leads?1 | Send Email Report1 | ## Step 3 – Report & Store<br>Leads are classified by confidence, emailed as a daily report, and saved to Google Sheets for long-term tracking. |
| Send Email Report1 | Gmail | Send HTML report email | Build Email Report1 | Merge1 | ## Step 3 – Report & Store<br>Leads are classified by confidence, emailed as a daily report, and saved to Google Sheets for long-term tracking. |
| Prepare Sheet Data1 | Code | Map enriched leads to sheet row schema | High Confidence Leads?1 | Save to Google Sheets | ## Step 3 – Report & Store<br>Leads are classified by confidence, emailed as a daily report, and saved to Google Sheets for long-term tracking. |
| Save to Google Sheets | Google Sheets | Append/update enriched leads in sheet | Prepare Sheet Data1 | Merge1; Format Error Data1 | ## Step 3 – Report & Store<br>Leads are classified by confidence, emailed as a daily report, and saved to Google Sheets for long-term tracking. |
| Merge1 | Merge | Synchronize report + save branches | Send Email Report1; Save to Google Sheets | Wait1 | ## Step 3 – Report & Store<br>Leads are classified by confidence, emailed as a daily report, and saved to Google Sheets for long-term tracking. |
| Wait1 | Wait | Loop control / pacing between batches | Merge1 | Batch for AI Processing1 | ## Step 3 – Report & Store<br>Leads are classified by confidence, emailed as a daily report, and saved to Google Sheets for long-term tracking. |
| Format Error Data1 | Code | Normalize errors for notification | Google Maps Scraper (error); Format & Validate Data (error); HTTP Request1 (error); Save to Google Sheets (error) | Error Notification1 |  |
| Error Notification1 | Gmail | Send error alert email | Format Error Data1 | — |  |
| Sticky Note4 | Sticky Note | Documentation block | — | — | ## Google Maps Lead Scraper with Enrichment & Email Reporting<br>This workflow is an automated lead generation system that scrapes businesses from Google Maps, enriches them with verified emails, and delivers a daily lead report while saving everything to Google Sheets.<br><br>### How it works<br>Step 1: The workflow runs on a daily schedule and scrapes businesses from Google Maps using a search query and location. The data is cleaned, normalized, scored for quality, and deduplicated against existing Google Sheets records.<br><br>Step 2: New leads are processed in small batches and enriched using Hunter domain search to find verified business emails. Emails are classified as High, Medium, or Low confidence.<br><br>Step 3: Leads are grouped by confidence level, a detailed HTML report is generated, and the results are emailed to you. All enriched leads are saved to Google Sheets for tracking.<br><br>### Setup steps<br>1. Connect Google Sheets and select the document for storing leads<br>2. Add your Apify API key in the Google Maps Scraper node<br>3. Add your Hunter API key for email enrichment<br>4. Connect Gmail for reports and notifications<br>5. Update the search query and location if needed<br>6. Turn the workflow ON and run once to test |
| Sticky Note5 | Sticky Note | Documentation block | — | — | ## Step 1 – Scrape & Deduplicate<br>Runs on schedule, scrapes Google Maps businesses, cleans and scores data, and removes duplicates already stored in Google Sheets. |
| Sticky Note6 | Sticky Note | Documentation block | — | — | ## Step 2 – Enrich Emails<br>New leads are processed in batches and enriched using Hunter domain search to find and verify business emails. |
| Sticky Note7 | Sticky Note | Documentation block | — | — | ## Step 3 – Report & Store<br>Leads are classified by confidence, emailed as a daily report, and saved to Google Sheets for long-term tracking. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Schedule Trigger**
   2. Set rule: run daily at **09:00** (or your preferred hour).

2) **Scrape Google Maps via Apify**
   1. Add node: **HTTP Request** named “Google Maps Scraper”
   2. Method: **POST**
   3. URL: `https://api.apify.com/v2/acts/<your-actor-or-account-id>/run-sync-get-dataset-items`
   4. Headers:
      - `Accept: application/json`
      - `Authorization: Bearer <APIFY_TOKEN>`
   5. Body (JSON):
      - `searchStringsArray`: include your `searchString` and `location`
      - `maxCrawledPlacesPerSearch`: set (e.g., 20)
   6. Options: set timeout to **300000 ms**
   7. Enable “Continue on Fail” / route error output (matching your n8n node capability).

3) **Normalize, score, and filter**
   1. Add **Code** node “Format & Validate Data”
   2. Paste logic to:
      - Detect the businesses array (`items` or direct array)
      - Normalize fields (name, address, phone, website, url, categories, rating, reviews, lat/lng)
      - Compute `data_quality_score`
      - Filter for score >= 40 and required fields
   3. Set node to continue error output (or handle with try/catch and manual branching).

4) **Read existing leads from Google Sheets**
   1. Add **Google Sheets** node “Check Existing Leads1”
   2. Credentials: connect Google account (OAuth2) with access to the sheet
   3. Select the **Spreadsheet** and the **Sheet** containing stored leads
   4. Configure to read rows (typical “Read” operation).
   5. Enable continue on fail (optional but matches current behavior).

5) **Deduplicate**
   1. Add **Code** node “Deduplicate Leads1”
   2. Implement:
      - Load new leads from “Format & Validate Data”
      - Load existing rows from “Check Existing Leads1”
      - Create Set key `business|website` (case-insensitive)
      - Filter uniques; if none, output a `{message: ...}` summary item

6) **Route when no new leads**
   1. Add **IF** node “Has New Leads?1”
   2. Condition: `$json.message` **is not equal** to “No new unique leads found”
   3. False branch → add **Gmail** node “No New Leads Notification1”
      - Configure Gmail OAuth2
      - To/Subject/Body using the dedupe summary variables

7) **Batch processing (Hunter enrichment)**
   1. True branch → add **Split in Batches** “Batch for AI Processing1” with batch size **5**
   2. Add **HTTP Request** “HTTP Request1” to call Hunter:
      - URL: `https://api.hunter.io/v2/domain-search`
      - Query params:
        - `domain`: extract from website (strip protocol and path)
        - `limit`: 5
      - Header: `X-Api-Key: <HUNTER_API_KEY>`
      - Enable continue-on-error output
   3. Add **Code** node “Merge & Validate Results”:
      - Merge Hunter results with original lead by index
      - Determine best email and map confidence to High/Medium/Low
      - Validate email format with regex
      - Attach `_errors` list to first output item if needed

   **Important:** To make batching reliable, connect **SplitInBatches output 0** to the Hunter HTTP Request, then after saving/reporting, loop back to SplitInBatches to fetch the next batch. The current JSON wiring uses output 1, which is atypical and can break processing depending on n8n behavior.

8) **Classify leads by confidence**
   1. Add **IF** node “High Confidence Leads?1”
   2. Condition: `email_confidence == "High"` AND email exists (prefer “is not empty” over comparing to string `"null"`).

9) **Generate and send HTML report**
   1. Add **Code** node “Build Email Report1”
      - Use `$items("Merge & Validate Results")` for totals
      - Use `$items("High Confidence Leads?1", 0/1)` for grouping
      - Build HTML and output as `email_html`
   2. Add **Gmail** node “Send Email Report1”
      - To: your address
      - Subject: include totals and date formatting
      - Body: `{{$json.email_html}}` (ensure HTML support)

10) **Prepare rows and save to Google Sheets**
   1. Add **Code** node “Prepare Sheet Data1” to map enriched leads to columns.
   2. Add **Google Sheets** node “Save to Google Sheets”
      - Operation: **Append or Update**
      - Map fields to the sheet headers
      - Set matching columns (prefer Business + Website/domain to avoid collisions)
      - Enable continue-on-error output

11) **Join branches and loop**
   1. Add **Merge** node to join “Send Email Report1” and “Save to Google Sheets”
   2. Add **Wait** node (configure explicitly—delay or webhook resume depending on your intent)
   3. Connect Wait → SplitInBatches to process next batch until finished

12) **Centralized error email**
   1. Add **Code** node “Format Error Data1” connected from error outputs
   2. Add **Gmail** node “Error Notification1” to email the formatted error details

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Google Maps Lead Scraper with Enrichment & Email Reporting” + setup steps (Google Sheets, Apify key, Hunter key, Gmail, update query/location, enable workflow) | Sticky Note content embedded in workflow |
| Step 1 – Scrape & Deduplicate | Sticky Note content embedded in workflow |
| Step 2 – Enrich Emails | Sticky Note content embedded in workflow |
| Step 3 – Report & Store | Sticky Note content embedded in workflow |

---