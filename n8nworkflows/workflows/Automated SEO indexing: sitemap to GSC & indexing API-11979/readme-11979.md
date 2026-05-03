Automated SEO indexing: sitemap to GSC & indexing API

https://n8nworkflows.xyz/workflows/automated-seo-indexing--sitemap-to-gsc---indexing-api-11979


# Automated SEO indexing: sitemap to GSC & indexing API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow automates SEO indexing operations by reading URLs from a sitemap (including sitemap indexes), filtering to recently modified URLs, checking each URL’s index status via **Google Search Console URL Inspection API**, and submitting eligible, non-indexed URLs to the **Google Indexing API** (URL_UPDATED).

**Target use cases:**
- Continuous/recurring indexing support for sites with frequent updates.
- Prioritizing recent content (last 90 days) and avoiding redundant resubmissions (7-day cool-down).
- Rate-limited, API-safe indexing requests.

### Logical blocks
1. **1.1 Triggers & Configuration**: manual/scheduled start + central parameters (sitemap URL, GSC property).
2. **1.2 Sitemap Retrieval & Parsing**: fetch XML, convert to JSON, detect sitemap index vs URL set.
3. **1.3 URL Normalization & Recency Filtering**: extract URL + lastmod; keep only recent.
4. **1.4 Deduping / Submission Decision**: check static submission history and decide whether to proceed.
5. **1.5 GSC Inspection & Indexing Decision**: inspect in GSC, determine if indexing is needed.
6. **1.6 Indexing API Submission with Rate Limiting**: batch loop, submit to Indexing API, wait between calls.

---

## 2. Block-by-Block Analysis

### 2.1 Triggers & Configuration
**Overview:** Starts the workflow either manually or on a weekly schedule, and defines the sitemap and GSC property used downstream.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Schedule Trigger
- Configuration
- Sticky Note (SETUP & CONFIGURATION)
- Sticky Note3 (Overview + setup steps)

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger; entry point for ad-hoc runs.
- **Config:** No parameters.
- **Connections:** Outputs to **Configuration**.
- **Failure/edge cases:** None.

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger; automated entry point.
- **Config:** Runs on an interval of **weeks** (weekly). (No specific weekday/time defined in the JSON.)
- **Connections:** Outputs to **Configuration**.
- **Failure/edge cases:** Timezone differences (n8n instance timezone), missed runs if instance is down.

#### Node: Configuration
- **Type / role:** Set node; centralizes workflow parameters.
- **Config choices:**
  - `sitemap`: `https://gamelandvn.com/sitemap_index.xml`
  - `gsc_property`: `https://gamelandvn.com/` (note trailing slash; important for URL-prefix properties)
- **Key expressions/variables:** None (static values).
- **Connections:** Outputs to **Fetch Sitemap XML**.
- **Failure/edge cases:** Wrong property URL format (missing trailing slash), incorrect sitemap URL, non-public sitemap.

#### Sticky Note: “# SETUP & CONFIGURATION”
- **Role:** Visual grouping only; no execution impact.

#### Sticky Note3: Overview + Setup steps (contains links and credential/scopes guidance)
- **Role:** Operational instructions and prerequisites. Key items:
  - Enable **Google Search Console API** + **Web Search Indexing API** in Google Cloud.
  - Use **Google Service Account** credential in n8n.
  - Required scopes:  
    `https://www.googleapis.com/auth/indexing`  
    `https://www.googleapis.com/auth/webmasters.readonly`
  - Add service account email as **Owner** in the GSC property.
  - Contact email: `admin@hanthienhai.com`
- **Link:** https://console.cloud.google.com/

---

### 2.2 Sitemap Retrieval & Parsing
**Overview:** Downloads the sitemap XML (or sitemap index XML), converts it into JSON, and emits either child sitemap URLs (if index) or page URLs (if single sitemap).

**Nodes involved:**
- Fetch Sitemap XML
- Convert XML to JSON
- Parse Sitemap Structure
- Is Sitemap Index?

#### Node: Fetch Sitemap XML
- **Type / role:** HTTP Request; retrieves XML.
- **Config choices:**
  - **URL:** `={{ $json.url || $node["Configuration"].json.sitemap }}`
    - If the current item already has a `url` (from sitemap index expansion), it fetches that; otherwise uses `Configuration.sitemap`.
- **Connections:** Input from **Configuration** or from **Is Sitemap Index? (true branch)**; output to **Convert XML to JSON**.
- **Failure/edge cases:**
  - 403/404, Cloudflare blocks, redirects, gzip issues.
  - Large sitemaps can cause memory/time issues.
  - If a child sitemap URL is invalid, parsing will fail downstream.

#### Node: Convert XML to JSON
- **Type / role:** XML node; parses sitemap XML into JSON.
- **Config choices:** trim/normalize enabled; attributes merged and not ignored; tags normalized.
- **Connections:** Input from **Fetch Sitemap XML**; output to **Parse Sitemap Structure**.
- **Failure/edge cases:** Non-XML response (HTML error page), malformed XML, unexpected namespaces.

#### Node: Parse Sitemap Structure
- **Type / role:** Code node; detects sitemap format and emits normalized items.
- **Logic (interpreted):**
  - If `sitemapindex.sitemap` exists: output items like `{ url: <child sitemap loc>, isSitemapIndex: true }`
  - Else if `urlset.url` exists: output items like `{ urlData: <url object>, isSitemapIndex: false }`
  - Handles array vs single object.
- **Connections:** Input from **Convert XML to JSON**; output to **Is Sitemap Index?**
- **Failure/edge cases:**
  - If XML-to-JSON structure differs (namespaces/alternate keys), emits empty output (workflow appears to do nothing).
  - If `<loc>` is missing, downstream fetch/format fails.

#### Node: Is Sitemap Index?
- **Type / role:** IF node; routes sitemap-index vs URL list.
- **Condition:** `={{ $json.isSitemapIndex }}` is true.
- **Connections:**
  - **True branch** → **Fetch Sitemap XML** (recursive expansion of child sitemaps)
  - **False branch** → **Format URL Data**
- **Failure/edge cases:** Mis-detection leads to wrong branch; if a sitemap index points to another index repeatedly, recursion could be large (though it only follows what the sitemap returns).

---

### 2.3 URL Normalization & Recency Filtering
**Overview:** Extracts standard fields (url, lastmod) from sitemap URL entries and keeps only those modified within the last 90 days.

**Nodes involved:**
- Format URL Data
- Filter: Recent URLs Only
- Sticky Note1 (DATA FILTERING & LOGIC)

#### Node: Format URL Data
- **Type / role:** Set node; transforms `urlData` into flat fields.
- **Config choices:**
  - `url`: `={{ $json.urlData.loc }}`
  - `lastmod`: `={{ $json.urlData.lastmod || '' }}`
  - `currentDate`: `={{ $now.toISO() }}` (not used later)
- **Connections:** Input from **Is Sitemap Index? (false branch)**; output to **Filter: Recent URLs Only**.
- **Failure/edge cases:** Missing `urlData.loc` produces empty/undefined URL; `lastmod` formats may not parse as date later.

#### Node: Filter: Recent URLs Only
- **Type / role:** Filter node; keeps only recent, well-formed candidates.
- **Conditions:**
  - `lastmod` is not empty
  - `lastmod` is after `={{ $now.minus({ days: 90 }).toISO() }}`
- **Connections:** Output to **Check Submission History**.
- **Failure/edge cases:**
  - If sitemap doesn’t provide `<lastmod>`, everything is filtered out.
  - If `lastmod` is not ISO-parseable, date comparison may fail (and items may be dropped).

#### Sticky Note1: “# DATA FILTERING & LOGIC”
- Visual grouping.

---

### 2.4 Deduping / Submission Decision
**Overview:** Uses n8n global static data to track prior submissions per URL and blocks resubmission if it was submitted within the last 7 days.

**Nodes involved:**
- Check Submission History
- Filter: Valid for Submission

#### Node: Check Submission History
- **Type / role:** Code node; determines `shouldSubmit` and `reason`.
- **Config choices / logic (interpreted):**
  - Builds a storage key: `indexed:<base64(url) first 50 chars>`
  - Reads `global` workflow static data via `$execution.getWorkflowStaticData('global')`
  - If URL exists in storage:
    - Computes days since last submission
    - Blocks submission if `< 7 days` (`shouldSubmit=false`)
    - Otherwise allows resubmit
  - On any error: defaults to submit with `reason='storage_error'`
- **Connections:** Output to **Filter: Valid for Submission**.
- **Failure/edge cases:**
  - **Important:** This node only *reads* static data; the provided workflow does **not** include a node that writes back “submitted timestamp” after successful indexing. Without an explicit write/update step, the history may never persist (depending on whether other unseen logic exists). As shown, deduping may be ineffective.
  - Static data persistence depends on n8n mode (and can be affected by restarts/DB issues).

#### Node: Filter: Valid for Submission
- **Type / role:** Filter node; keeps only `shouldSubmit === true`.
- **Condition:** `={{ $json.shouldSubmit }}` is true.
- **Connections:** Output to **GSC: Inspect URL Status**.
- **Failure/edge cases:** If previous node output is missing `shouldSubmit`, items may be filtered out unexpectedly.

---

### 2.5 GSC Inspection & Indexing Decision
**Overview:** Calls the URL Inspection API for each URL and decides whether the URL “needs indexing” based on coverage state.

**Nodes involved:**
- GSC: Inspect URL Status
- Logic: Should Index?
- Filter: Only Not Indexed
- Sticky Note2 (API INTERACTION & INDEXING)

#### Node: GSC: Inspect URL Status
- **Type / role:** HTTP Request; calls Google Search Console URL Inspection API.
- **Request:**
  - POST `https://searchconsole.googleapis.com/v1/urlInspection/index:inspect`
  - JSON body:
    - `inspectionUrl`: `{{ $json.url }}`
    - `siteUrl`: `{{ $('Configuration').item.json.gsc_property }}`
- **Auth:** Predefined credential type `googleApi` (service account), scopes required per sticky note.
- **Options:** Batching enabled with batchSize=1 and batchInterval=2000ms (effectively rate limiting / spacing).
- **continueOnFail:** true (workflow continues even if API call fails).
- **Connections:** Output to **Logic: Should Index?**
- **Failure/edge cases:**
  - 403 if service account not added to GSC property.
  - 400 if `siteUrl` mismatch (wrong property type/format).
  - Quotas/rate limits; partial failures because continueOnFail is enabled (downstream must handle error-shaped items).

#### Node: Logic: Should Index?
- **Type / role:** Code node; converts GSC response into a simple decision.
- **Logic (interpreted):**
  - Default: `needsIndexing=true`, `indexStatus='unknown'`
  - If `inspectionResult.indexStatusResult.coverageState` is:
    - `Submitted and indexed` OR `Indexed, not submitted in sitemap` → `needsIndexing=false`
  - On error: `indexStatus='check_failed'` and keeps `needsIndexing=true`
  - Outputs:
    - `url`
    - `needsIndexing`
    - `indexStatus`
    - `originalData` (entire prior JSON)
- **Connections:** Output to **Filter: Only Not Indexed**
- **Failure/edge cases:**
  - If upstream call failed, structure may differ; the code catches broadly and may mark it as needing indexing (could cause unnecessary submissions).
  - Coverage state strings may vary by API changes/localization; strict string match is brittle.

#### Node: Filter: Only Not Indexed
- **Type / role:** Filter node; keeps only URLs needing indexing and with a non-unspecified indexing state.
- **Conditions:**
  - `needsIndexing` is true
  - `originalData.inspectionResult.indexStatusResult.indexingState` != `INDEXING_STATE_UNSPECIFIED`
- **Connections:** Output to **Batch Processing**
- **Failure/edge cases:**
  - If `originalData.inspectionResult...` is missing (due to failed inspection), expression may evaluate to null/undefined; filtering behavior may exclude or include unexpectedly.
  - This condition can unintentionally drop items if the API returns `INDEXING_STATE_UNSPECIFIED` for cases you still want to submit.

#### Sticky Note2: “# API INTERACTION & INDEXING”
- Visual grouping.

---

### 2.6 Indexing API Submission with Rate Limiting
**Overview:** Sends URLs to Google Indexing API one by one via a batch loop and waits 2 seconds between requests.

**Nodes involved:**
- Batch Processing
- GSC: Request Indexing
- Delay (Rate Limiting)

#### Node: Batch Processing
- **Type / role:** Split In Batches; iterates over items.
- **Config:** Default batch settings (batch size not explicitly set in JSON).
- **Connections:**
  - **Output 1** (main batch output) is unused in connections shown.
  - **Output 2** (continue output) → **GSC: Request Indexing**
  - After **Delay**, it loops back into **Batch Processing** to continue.
- **Failure/edge cases:**
  - Mis-wiring risk: in this workflow, the connected output is the “continue” branch. This can work, but it’s easy to break if batch configuration changes.
  - If no items, loop ends immediately.

#### Node: GSC: Request Indexing
- **Type / role:** HTTP Request; calls Google Indexing API publish endpoint.
- **Request:**
  - POST `https://indexing.googleapis.com/v3/urlNotifications:publish`
  - JSON body:
    - `url`: `{{ $json.originalData.inspectionResult.indexStatusResult.userCanonical }}`
    - `type`: `URL_UPDATED`
- **Auth:** `googleApi` credential (same as inspection).
- **continueOnFail:** true
- **alwaysOutputData:** true (ensures an item continues even on failure)
- **Connections:** Output to **Delay (Rate Limiting)**.
- **Failure/edge cases:**
  - If `userCanonical` is missing, request submits null/empty URL → 400.
  - Indexing API is officially intended for certain content types (e.g., JobPosting, BroadcastEvent). Using it for general URLs can lead to rejections or ineffectiveness.
  - Quota/rate limiting; continueOnFail may hide systematic failures unless you log results.

#### Node: Delay (Rate Limiting)
- **Type / role:** Wait node; enforces a pause between requests.
- **Config:** 2 seconds.
- **Connections:** Output back to **Batch Processing** (loop continuation).
- **Failure/edge cases:** Prolongs execution time; on large sets, workflow may exceed n8n execution time limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Configuration | # SETUP & CONFIGURATION |
| Schedule Trigger | Schedule Trigger | Weekly automation entry point | — | Configuration | # SETUP & CONFIGURATION |
| Configuration | Set | Stores sitemap + GSC property parameters | When clicking ‘Execute workflow’; Schedule Trigger | Fetch Sitemap XML | # SETUP & CONFIGURATION |
| Fetch Sitemap XML | HTTP Request | Downloads sitemap XML (index or child) | Configuration; Is Sitemap Index? | Convert XML to JSON | # SETUP & CONFIGURATION |
| Convert XML to JSON | XML | Parses sitemap XML into JSON | Fetch Sitemap XML | Parse Sitemap Structure | # SETUP & CONFIGURATION |
| Parse Sitemap Structure | Code | Detects sitemapindex vs urlset; emits items | Convert XML to JSON | Is Sitemap Index? | # SETUP & CONFIGURATION |
| Is Sitemap Index? | IF | Branch: expand child sitemaps vs process URLs | Parse Sitemap Structure | Fetch Sitemap XML (true); Format URL Data (false) | # SETUP & CONFIGURATION |
| Format URL Data | Set | Extracts url/lastmod fields | Is Sitemap Index? (false) | Filter: Recent URLs Only | # DATA FILTERING & LOGIC |
| Filter: Recent URLs Only | Filter | Keeps URLs with lastmod within last 90 days | Format URL Data | Check Submission History | # DATA FILTERING & LOGIC |
| Check Submission History | Code | Deduping logic via global static data; sets shouldSubmit | Filter: Recent URLs Only | Filter: Valid for Submission | # DATA FILTERING & LOGIC |
| Filter: Valid for Submission | Filter | Allows only shouldSubmit=true | Check Submission History | GSC: Inspect URL Status | # API INTERACTION & INDEXING |
| GSC: Inspect URL Status | HTTP Request | URL Inspection API call | Filter: Valid for Submission | Logic: Should Index? | # API INTERACTION & INDEXING |
| Logic: Should Index? | Code | Determines needsIndexing based on inspection coverageState | GSC: Inspect URL Status | Filter: Only Not Indexed | # API INTERACTION & INDEXING |
| Filter: Only Not Indexed | Filter | Keeps only URLs that need indexing | Logic: Should Index? | Batch Processing | # API INTERACTION & INDEXING |
| Batch Processing | Split In Batches | Iterates items for controlled submissions | Filter: Only Not Indexed; Delay (Rate Limiting) | GSC: Request Indexing (continue output) | # API INTERACTION & INDEXING |
| GSC: Request Indexing | HTTP Request | Sends URL_UPDATED notification to Indexing API | Batch Processing | Delay (Rate Limiting) | # API INTERACTION & INDEXING |
| Delay (Rate Limiting) | Wait | 2s pause between submissions | GSC: Request Indexing | Batch Processing | # API INTERACTION & INDEXING |
| Sticky Note | Sticky Note | Section label | — | — | # SETUP & CONFIGURATION |
| Sticky Note1 | Sticky Note | Section label | — | — | # DATA FILTERING & LOGIC |
| Sticky Note2 | Sticky Note | Section label | — | — | # API INTERACTION & INDEXING |
| Sticky Note3 | Sticky Note | Setup/prereqs text and links | — | — | ## Overview (includes Google Cloud link and setup instructions) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: `Automated SEO Indexing: Sitemap to GSC & Indexing API`.

2. **Add triggers**
   1. Add **Manual Trigger** node named `When clicking ‘Execute workflow’`.
   2. Add **Schedule Trigger** node named `Schedule Trigger`.
      - Set interval to **Weeks** (weekly). Optionally define the exact day/time in UI.

3. **Add configuration node**
   1. Add **Set** node named `Configuration`.
   2. Add fields:
      - `sitemap` (String): your sitemap index URL (e.g., `https://example.com/sitemap_index.xml`)
      - `gsc_property` (String): your Search Console property URL  
        - If your property is **URL prefix**, ensure it ends with `/` (e.g., `https://example.com/`).

4. **Connect triggers to configuration**
   - Connect **Manual Trigger → Configuration**
   - Connect **Schedule Trigger → Configuration**

5. **Sitemap fetch + parse**
   1. Add **HTTP Request** node named `Fetch Sitemap XML`.
      - Method: GET
      - URL: `={{ $json.url || $node["Configuration"].json.sitemap }}`
   2. Add **XML** node named `Convert XML to JSON`.
      - Enable: trim, normalize, merge attributes, normalize tags (matching the workflow settings).
   3. Add **Code** node named `Parse Sitemap Structure` with code implementing:
      - If sitemap index: emit `{ url: loc, isSitemapIndex: true }`
      - If urlset: emit `{ urlData: urlObject, isSitemapIndex: false }`
   4. Add **IF** node named `Is Sitemap Index?`
      - Condition: Boolean `={{ $json.isSitemapIndex }}` is true
   5. Connect:
      - `Configuration → Fetch Sitemap XML → Convert XML to JSON → Parse Sitemap Structure → Is Sitemap Index?`
      - From **Is Sitemap Index? (true)** connect back to **Fetch Sitemap XML** (to fetch child sitemaps).
      - From **Is Sitemap Index? (false)** continue to the next block.

6. **Normalize URL fields**
   1. Add **Set** node named `Format URL Data`.
      - `url` = `={{ $json.urlData.loc }}`
      - `lastmod` = `={{ $json.urlData.lastmod || '' }}`
      - `currentDate` = `={{ $now.toISO() }}`
   2. Connect **Is Sitemap Index? (false) → Format URL Data**.

7. **Filter to recent URLs**
   1. Add **Filter** node named `Filter: Recent URLs Only`.
      - Condition 1: String `lastmod` is not empty
      - Condition 2: DateTime `lastmod` is after `={{ $now.minus({ days: 90 }).toISO() }}`
   2. Connect **Format URL Data → Filter: Recent URLs Only**.

8. **Check submission history (dedupe)**
   1. Add **Code** node named `Check Submission History`.
      - Implement:
        - Read global static data
        - Determine `shouldSubmit` with a 7-day cool-down
        - Output `shouldSubmit` and `reason`
   2. Add **Filter** node named `Filter: Valid for Submission`.
      - Condition: Boolean `shouldSubmit` is true
   3. Connect:
      - `Filter: Recent URLs Only → Check Submission History → Filter: Valid for Submission`

9. **Configure Google credentials (required)**
   1. In **Google Cloud Console**:
      - Create a project
      - Enable **Google Search Console API** and **Web Search Indexing API**
      - Create a **Service Account** and generate a JSON key
   2. In n8n **Credentials**:
      - Create **Google Service Account API** (used as `googleApi` in HTTP Request nodes)
      - Paste `client_email` and `private_key`
      - Enable “Set up for use in HTTP Request node”
      - Scopes (space-separated):  
        `https://www.googleapis.com/auth/indexing https://www.googleapis.com/auth/webmasters.readonly`
   3. In **Google Search Console**:
      - Add the service account email as **Owner** on the property referenced in `gsc_property`.

10. **Inspect URLs in GSC**
    1. Add **HTTP Request** node named `GSC: Inspect URL Status`.
       - Method: POST
       - URL: `https://searchconsole.googleapis.com/v1/urlInspection/index:inspect`
       - Authentication: predefined credential type `googleApi` (service account)
       - Body type: JSON
       - Body:
         - `inspectionUrl`: `{{ $json.url }}`
         - `siteUrl`: `{{ $('Configuration').item.json.gsc_property }}`
       - Options: batching batchSize=1, batchInterval=2000ms
       - Enable **Continue On Fail**
    2. Add **Code** node named `Logic: Should Index?` to set:
       - `needsIndexing` false if coverageState indicates already indexed
       - Output `originalData`
    3. Add **Filter** node named `Filter: Only Not Indexed`:
       - `needsIndexing` is true
       - `originalData.inspectionResult.indexStatusResult.indexingState` != `INDEXING_STATE_UNSPECIFIED`
    4. Connect:
       - `Filter: Valid for Submission → GSC: Inspect URL Status → Logic: Should Index? → Filter: Only Not Indexed`

11. **Submit to Indexing API with pacing**
    1. Add **Split In Batches** node named `Batch Processing`.
    2. Add **HTTP Request** node named `GSC: Request Indexing`.
       - Method: POST
       - URL: `https://indexing.googleapis.com/v3/urlNotifications:publish`
       - Authentication: `googleApi`
       - Body JSON:
         - `url`: `{{ $json.originalData.inspectionResult.indexStatusResult.userCanonical }}`
         - `type`: `URL_UPDATED`
       - Enable **Continue On Fail**
       - Enable **Always Output Data**
    3. Add **Wait** node named `Delay (Rate Limiting)`:
       - Unit: seconds
       - Amount: 2
    4. Connect:
       - `Filter: Only Not Indexed → Batch Processing`
       - `Batch Processing (continue output) → GSC: Request Indexing → Delay (Rate Limiting) → Batch Processing` (loop)

12. **(Recommended but missing in current workflow) Persist successful submissions**
    - Add a Code node after `GSC: Request Indexing` to write the current timestamp into global static data for the URL key, so the 7-day dedupe actually works on future runs.

13. **Activate workflow**
    - Turn workflow **Active** to allow the Schedule Trigger to run.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Cloud Console project setup is required; enable Search Console API + Web Search Indexing API. | https://console.cloud.google.com/ |
| Service account scopes must include: `https://www.googleapis.com/auth/indexing` and `https://www.googleapis.com/auth/webmasters.readonly` | Credential configuration note (Sticky Note3) |
| Add service account email as **Owner** of the GSC property. | GSC Settings → Users and permissions |
| Configuration: if GSC property is URL-prefix, it must end with `/` (example: `https://hanthienhai.com/`). | Sticky Note3 |
| Support contact mentioned in the workflow note: `admin@hanthienhai.com` | Sticky Note3 |