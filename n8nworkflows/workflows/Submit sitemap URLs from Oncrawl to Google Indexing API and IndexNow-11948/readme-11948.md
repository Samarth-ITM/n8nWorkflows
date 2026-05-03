Submit sitemap URLs from Oncrawl to Google Indexing API and IndexNow

https://n8nworkflows.xyz/workflows/submit-sitemap-urls-from-oncrawl-to-google-indexing-api-and-indexnow-11948


# Submit sitemap URLs from Oncrawl to Google Indexing API and IndexNow

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Submit sitemap URLs from Oncrawl to Google Indexing API and IndexNow  
**Workflow name (in n8n):** 🏁 Submit URLs for IndexNow & Google Indexing API

**Purpose:**  
This workflow receives an Oncrawl “crawl finished” notification (webhook callback), discovers the sitemap URL(s) declared in robots.txt via Oncrawl, parses the sitemap index and underlying sitemaps, filters URLs updated recently (based on `DAYS_BACK`), then submits those URLs:
- in **batches to IndexNow** (max 500 URLs per request), and/or
- **one-by-one to Google Indexing API**, after checking each URL’s current Search Console inspection status and recency.

**Target use cases:**
- Fast indexation of recently updated business-critical pages.
- Minimizing API rate limiting by filtering recent URLs and adding jitter waits.
- Optional “variations” (additional branches) to index orphan pages or crawl-over-crawl deltas (present in workflow but not fully wired into the main path).

### 1.1 Input Reception & Sitemap Discovery (Oncrawl)
Receives the webhook, calls Oncrawl “discover_sitemaps”, stores configuration.

### 1.2 Sitemap Parsing & URL List Preparation
Fetches sitemap index, iterates content-specific sitemaps, normalizes XML-to-JSON structure, extracts URL entries, sorts by `lastmod`, and standardizes required fields (`loc`, `lastmod`).

### 1.3 Freshness Filter (DAYS_BACK)
Filters URLs to only those whose `lastmod` is within the last N days to reduce volume and rate-limit risk.

### 1.4 IndexNow Submission (batched)
If enabled, batches URLs (≤500), builds IndexNow payload (host/key/keyLocation/urlList), submits to IndexNow with jitter.

### 1.5 Google Indexing API Submission (per-URL)
If enabled, loops through URLs, checks Search Console URL Inspection status, submits `URL_UPDATED` when needed, and waits between calls.

### 1.6 Optional Variations (present, not fully connected end-to-end)
- Variation A: Orphan pages via Oncrawl Data API + merge to recover `lastmod`.
- Variation B: Newly added pages between Crawl1 & Crawl2 via Crawl-over-Crawl API + merge to recover `lastmod`.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Reception & Oncrawl Sitemap Discovery
**Overview:** Receives Oncrawl notification when a crawl ends, then requests Oncrawl to discover sitemaps from robots.txt and initializes runtime configuration variables.

**Nodes involved:**
- Webhook
- Post - Get Sitemaps
- Config
- Sticky Note2
- Sticky Note3
- Sticky Note
- Sticky Note7

#### Node: Webhook
- **Type / role:** `Webhook` (entry point). Receives POST callbacks.
- **Config choices:**
  - HTTP Method: POST
  - Response mode: **responseNode** (workflow should respond via a response node; however there is no explicit Response node in this JSON—see edge cases).
  - Path: `97a1f054-d877-4677-9c79-f10b0092e086`
- **Key variables/fields expected (from Oncrawl notification):**
  - `$json.body.project.id`
  - `$json.body.crawl.id`
  - `$json.body.crawl.configuration.name`
- **Connections:**
  - Output → **Post - Get Sitemaps**
- **Edge cases / failures:**
  - If Oncrawl payload structure differs, expressions referencing `$json.body...` will fail downstream.
  - Since responseMode is `responseNode` but no Response node exists, executions may hang or return default behavior depending on n8n version/settings.

#### Node: Post - Get Sitemaps
- **Type / role:** `HTTP Request` to Oncrawl API: creates a crawl config using `discover_sitemaps` endpoint to detect sitemaps declared in robots.txt.
- **Config choices:**
  - URL: `https://app.oncrawl.com/api/v2/projects/{{ $json.body.project.id }}/crawl_configs/discover_sitemaps`
  - Method: POST
  - Body: JSON (large crawl_config payload); important outputs are expected under `results[0].url`.
  - Header: Content-Type set to ` application/json` (note leading space).
- **Expressions:**
  - Uses `{{ $json.body.project.id }}` and `{{ $json.body.crawl.configuration.name }}` from Webhook payload.
- **Connections:**
  - Output → **Config**
- **Edge cases / failures:**
  - Missing/invalid Oncrawl auth (not shown in JSON; must be configured in credentials or headers).
  - Content-Type header has a leading space; some servers tolerate it, but it is safer to remove.
  - The body includes placeholders (`URL/`, `Email`, `Your_Webhook`, `Scenario`) that are not dynamically set—if Oncrawl validates these strictly, request can fail.

#### Node: Config
- **Type / role:** `Set` node. Central configuration and toggles used across the workflow via `$items('Config')[0].json`.
- **Config choices (interpreted):**
  - Stores crawl id: `body.crawl.id` from Webhook.
  - Stores sitemap URL from Oncrawl discovery: `SITEMAP_URL = $json.results[0].url`
  - Defines operational parameters:
    - `DAYS_BACK` (default 7)
    - `BATCH_SIZE` (default 500)
    - `USE_GOOGLE` (default true)
    - `USE_INDEXNOW` (default true)
    - `SITE_URL` (must be filled)
    - `INDEXNOW_KEY` (must be filled)
    - `INDEXNOW_KEY_URL` (must be filled)
- **Connections:**
  - Output → **Get sitemap.xml**
- **Edge cases / failures:**
  - If `results[0].url` is missing, sitemap fetching fails.
  - If `SITE_URL` is blank, IndexNow payload builder will drop all URLs (host cannot be derived).
  - Credentials are not stored here; Google and Oncrawl auth must be configured on HTTP nodes.

**Sticky notes covering this block:**
- **Sticky Note2:** Explains SEO context, IndexNow vs Google importance, Oncrawl API token guidance, and mentions workflow variations + link: https://n8n.io/workflows/8778-workflow-for-submitting-changed-sitemap-urls-using-google-indexing-api-and-bing-indexnow/
- **Sticky Note3:** Webhook callback explanation + Oncrawl notification doc: https://developer.oncrawl.com/#notification
- **Sticky Note:** Discover_sitemaps endpoint explanation + doc link: https://developer.oncrawl.com/
- **Sticky Note7:** Config variables to update + IndexNow key instructions + Bing link: https://www.bing.com/indexnow/getstarted

---

### Block 2 — Sitemap Parsing, Normalization, Extraction, Sorting
**Overview:** Fetches the sitemap index, enumerates content-specific sitemaps, parses XML into JSON, normalizes `urlset.url` into an array, splits into individual URLs, sorts by `lastmod`, and standardizes required fields.

**Nodes involved:**
- Get sitemap.xml
- Convert sitemap to JSON
- Get content-specific sitemaps
- Get content of each sitemap
- convert page data to JSON
- Force urlset.url to array
- Split Out
- Sort
- Assign mandatory sitemap fields
- Sticky Note9

#### Node: Get sitemap.xml
- **Type / role:** `HTTP Request` to fetch sitemap index XML.
- **Config choices:**
  - URL: `{{ $json.SITEMAP_URL }}`
- **Connections:**
  - Output → **Convert sitemap to JSON**
- **Edge cases / failures:**
  - Non-200 response, redirects, gzip handling, blocked user-agent, etc.
  - If `SITEMAP_URL` points to a sitemap (not sitemapindex), next split may not match schema.

#### Node: Convert sitemap to JSON
- **Type / role:** `XML` node. Converts sitemap XML to JSON.
- **Config choices:** default conversion options.
- **Connections:**
  - Output → **Get content-specific sitemaps**
- **Edge cases:**
  - XML namespaces can alter parsed structure depending on n8n XML parser behavior.

#### Node: Get content-specific sitemaps
- **Type / role:** `Split Out`. Iterates entries under `sitemapindex.sitemap`.
- **Config choices:**
  - Field to split: `sitemapindex.sitemap`
- **Connections:**
  - Output → **Get content of each sitemap**
- **Edge cases:**
  - If the root is not `sitemapindex` (single sitemap), this field is missing and produces no items.

#### Node: Get content of each sitemap
- **Type / role:** `HTTP Request`. Fetches each sitemap URL (`loc`) from sitemap index.
- **Config choices:**
  - URL: `{{ $json.loc }}`
- **Connections:**
  - Output → **convert page data to JSON**
- **Edge cases:**
  - Individual sitemap may be large; timeouts possible.

#### Node: convert page data to JSON
- **Type / role:** `XML` node for URL sitemap content.
- **Config choices:**
  - `explicitArray: false` to simplify JSON structure when elements are singletons.
- **Connections:**
  - Output → **Force urlset.url to array**
- **Edge cases:**
  - Because `explicitArray` is false, `urlset.url` can be object or array; next node addresses this.

#### Node: Force urlset.url to array
- **Type / role:** `Set`. Normalizes `urlset.url` so downstream `Split Out` always works.
- **Key expression:**
  - `urlset.url = $json.urlset && $json.urlset.url ? ($json.urlset.url[0] ? $json.urlset.url : [$json.urlset.url]) : []`
- **Connections:**
  - Output → **Split Out**
- **Edge cases:**
  - If sitemap is empty or structure differs, results in empty array (safe).

#### Node: Split Out
- **Type / role:** `Split Out`. Splits URL entries.
- **Config choices:**
  - Field to split: `urlset.url`
- **Connections:**
  - Output → **Sort**

#### Node: Sort
- **Type / role:** `Sort`. Sorts by `lastmod` descending.
- **Config choices:**
  - Sort field: `lastmod` (descending)
- **Connections:**
  - Output → **Assign mandatory sitemap fields**
- **Edge cases:**
  - If `lastmod` missing or inconsistent, sort order may be unstable.

#### Node: Assign mandatory sitemap fields
- **Type / role:** `Set`. Ensures each item has `loc` and `lastmod` fields at top level.
- **Assignments:**
  - `lastmod = $json.lastmod`
  - `loc = $json.loc`
- **Connections:**
  - Output → **Filter: lastmod within DAYS_BACK**

**Sticky note:**
- **Sticky Note9:** “Parsing the sitemap and generating a list of the urls order by last modification date. No need to edit anything here.”

---

### Block 3 — Freshness Filter (DAYS_BACK)
**Overview:** Filters out URLs whose `lastmod` is older than `DAYS_BACK` days. This reduces submissions and helps prevent API rate limits.

**Nodes involved:**
- Filter: lastmod within DAYS_BACK
- Sticky Note1

#### Node: Filter: lastmod within DAYS_BACK
- **Type / role:** `Code`. Filters items by date.
- **Key variables/expressions:**
  - `const daysBack = $items('Config')[0].json.DAYS_BACK || 7;`
  - `cutoff = now - daysBack`
  - Keeps only items where `new Date(lastmod) >= cutoff`
  - Outputs simplified `{ loc, lastmod }`
- **Connections:**
  - Output → **Gate: IndexNow**
  - Output → **Gate: Google**
- **Edge cases / failures:**
  - If `lastmod` is missing or not parseable as a Date, item is dropped.
  - If `Config` is missing (renamed node, different path), `$items('Config')` breaks.

**Sticky note:**
- **Sticky Note1:** Notes rate-limit protection and cutoff calculation.

---

### Block 4 — IndexNow Submission Path (Batch ≤ 500)
**Overview:** If enabled (`USE_INDEXNOW`), batches filtered URLs, constructs a valid IndexNow payload, submits it, and waits with jitter between batches.

**Nodes involved:**
- Gate: IndexNow
- Split In Batches (IndexNow ≤500)1
- Build IndexNow payload
- IndexNow Submit
- Wait (IndexNow jitter)1
- Sticky Note8

#### Node: Gate: IndexNow
- **Type / role:** `Code`. Feature flag gate.
- **Logic:** If `Config.USE_INDEXNOW` is true (boolean true or string `"true"`), passes items through; else outputs `[]`.
- **Connections:**
  - Output → **Split In Batches (IndexNow ≤500)1**
- **Edge cases:**
  - If Config node renamed, `$items('Config')` fails.

#### Node: Split In Batches (IndexNow ≤500)1
- **Type / role:** `Split In Batches`. Controls request size.
- **Config choices:**
  - Batch size: `min(Config.BATCH_SIZE || 500, 500)` to enforce IndexNow max 500.
- **Connections:**
  - Main output (index 1) → **Build IndexNow payload**
  - After each batch completes, it loops (via Wait node returning to SplitInBatches).
- **Edge cases:**
  - If BATCH_SIZE is 0/invalid, could produce unexpected behavior; expression tries to fallback.

#### Node: Build IndexNow payload
- **Type / role:** `Code`. Builds IndexNow request body.
- **Key logic:**
  - Extract `loc` URLs, trim, remove empties
  - Keep only `http/https`
  - Derive `host` from `Config.SITE_URL` (regex `^https?://([^/]+)`)
  - Filter URLs to same host
  - De-duplicate
  - Guard: if no valid URLs or missing host ⇒ return `[]`
  - Output payload:
    - `host`
    - `key` (Config.INDEXNOW_KEY)
    - `keyLocation` (Config.INDEXNOW_KEY_URL)
    - `urlList` (array)
- **Config choices:**
  - `alwaysOutputData: true` (node returns data even if empty; but code returns `[]` in guards).
- **Connections:**
  - Output → **IndexNow Submit**
- **Edge cases / failures:**
  - If `SITE_URL` missing or not absolute, host extraction fails ⇒ nothing submitted.
  - If sitemap contains multiple hosts (cdn, http/https mismatch), those URLs are dropped.
  - If IndexNow key fields blank, API may reject with 4xx.

#### Node: IndexNow Submit
- **Type / role:** `HTTP Request`. Calls IndexNow endpoint.
- **Config choices:**
  - URL: `https://api.indexnow.org/indexnow`
  - Method: POST
  - JSON body: `={{ $json }}`
  - Response: full response enabled
  - `retryOnFail: false`
- **Connections:**
  - Output → **Wait (IndexNow jitter)1**
- **Edge cases / failures:**
  - 4xx on invalid key/keyLocation/host mismatch.
  - 429 rate limiting possible (jitter helps).

#### Node: Wait (IndexNow jitter)1
- **Type / role:** `Wait`. Adds random delay between batch submissions.
- **Config:**
  - Seconds: `(0.25 + Math.random()*0.75).toFixed(2)` → 0.25–1.00s
- **Connections:**
  - Output → **Split In Batches (IndexNow ≤500)1** (continue loop)
- **Edge cases:**
  - None major; ensures spacing.

**Sticky note:**
- **Sticky Note8:** Documents the IndexNow autosubmitting steps.

---

### Block 5 — Google URL Inspection + Indexing API Submission Path
**Overview:** If enabled (`USE_GOOGLE`), processes URLs one by one. It checks URL Inspection status in Search Console, submits `URL_UPDATED` for “Crawled - currently not indexed” and for already indexed pages whose sitemap `lastmod` is after Google’s last crawl time.

**Nodes involved:**
- Gate: Google
- Loop Over Items
- Check status
- Switch
- is new?
- URL Updated
- Wait2
- Sticky Note12

#### Node: Gate: Google
- **Type / role:** `Code`. Feature flag gate.
- **Logic:** Pass-through items if `Config.USE_GOOGLE` is true (boolean or `"true"`).
- **Connections:**
  - Output → **Loop Over Items**
- **Edge cases:** Same Config naming dependency.

#### Node: Loop Over Items
- **Type / role:** `Split In Batches`. Used here as an item-by-item loop.
- **Config:** options empty (defaults to batch size 1 in common usage; in n8n this node typically defaults to 1 unless set).
- **Connections:**
  - Output (index 1) → **Check status**
  - Output (index 0) is unused in connections (empty).
- **Edge cases:**
  - If batch size default changes or is not 1, logic may not behave as intended.

#### Node: Check status
- **Type / role:** `HTTP Request`. Calls Google Search Console URL Inspection API.
- **Config choices:**
  - URL: `https://searchconsole.googleapis.com/v1/urlInspection/index:inspect`
  - Method: POST
  - Content type: `form-urlencoded`
  - Body parameters:
    - `inspectionUrl = {{$json.loc}}`
    - `siteUrl` is present but **has no value configured** (likely must be set to Config.SITE_URL or Search Console property URL).
  - Response: full response enabled
  - `retryOnFail: false`
- **Connections:**
  - Output → **Switch**
- **Version/credential requirements:**
  - Requires Google OAuth2 credentials with Search Console scopes, or service account with delegated access depending on setup.
- **Edge cases / failures:**
  - If `siteUrl` parameter is empty, Google will return 400.
  - If property is not verified in Search Console, returns permission errors.
  - Rate limits can occur.

#### Node: Switch
- **Type / role:** `Switch`. Routes based on `coverageState`.
- **Rules:**
  1. If `coverageState == "Submitted and indexed"` → output 1 → **is new?**
  2. If `coverageState == "Crawled - currently not indexed"` → output 2 → **URL Updated**
- **Expression:**
  - `$json.body.inspectionResult.indexStatusResult.coverageState`
- **Connections:**
  - Output 0 → **is new?**
  - Output 1 → **URL Updated**
- **Edge cases:**
  - Other coverage states are not handled → item is dropped (no default output configured).

#### Node: is new?
- **Type / role:** `IF`. Checks whether sitemap lastmod is after Google last crawl.
- **Condition:**
  - `$('Loop Over Items').item.json.lastmod` **after** `$json.body.inspectionResult.indexStatusResult.lastCrawlTime`
- **Connections:**
  - True → **URL Updated**
  - False → **Wait2** (then loop continues without submission)
- **Edge cases:**
  - If `lastCrawlTime` is missing/null, comparison may fail or behave loosely (node uses loose validation).
  - Relies on referencing Loop Over Items context; if node names change, expression breaks.

#### Node: URL Updated
- **Type / role:** `HTTP Request`. Calls Google Indexing API publish endpoint.
- **Config choices:**
  - URL: `https://indexing.googleapis.com/v3/urlNotifications:publish`
  - Method: POST
  - Body parameters:
    - `url = {{ $('Loop Over Items').item.json.loc }}`
    - `type = URL_UPDATED`
- **Connections:**
  - Output → **Wait2**
- **Credential requirements:**
  - Google Indexing API enabled in GCP; OAuth/service account with correct scope and permissions.
- **Edge cases / failures:**
  - Indexing API is intended for specific content types (e.g., JobPosting, BroadcastEvent). For general web pages it may be rejected/ignored.
  - 403 if API not enabled or principal not authorized.

#### Node: Wait2
- **Type / role:** `Wait`. Random delay to reduce rate limiting.
- **Config:**
  - Seconds: `Math.min(1.5,0.3+3*Math.random()).toFixed(2)` → 0.30–1.50s
- **Connections:**
  - Output → **Loop Over Items** (continues loop)
- **Edge cases:** None major.

**Sticky note:**
- **Sticky Note12:** Documents the intended Google logic and the jitter wait.

---

### Block 6 — Variation A (Orphan Pages) (present, partial wiring)
**Overview:** Pulls orphan pages from Oncrawl Data API (based on OQL), splits nested arrays, then merges with sitemap-derived items to recover `lastmod`. This block is not connected from the main execution path in the provided connections.

**Nodes involved:**
- Config - Orphan
- Get Orphan Pages
- Split Out - Orphan1
- Split Out - Orphan2
- Merge
- Sticky Note10

#### Node: Config - Orphan
- **Type / role:** `Set`. Alternative config example populated with oncrawl.com values and an IndexNow key.
- **Connections:**
  - Output → **Get Orphan Pages**
- **Edge cases:**
  - This is a separate entry-like configuration but it is not connected to Webhook; running it standalone requires manual execution.

#### Node: Get Orphan Pages
- **Type / role:** `HTTP Request` to Oncrawl Data API pages endpoint.
- **Config:**
  - URL: `https://app.oncrawl.com/api/v2/data/crawl/{{ $json.body.crawl.id }}/pages`
  - Method: POST
  - Body: OQL that targets orphan pages (depth has no value) from sources logs_cross_analysis or sitemaps.
  - Fields: `url`, `sources`
- **Connections:**
  - Output → **Split Out - Orphan1**
- **Edge cases:**
  - Requires Oncrawl API auth.
  - Pagination fixed at limit 1000; additional pages ignored unless extended.

#### Node: Split Out - Orphan1
- **Type / role:** `Split Out` on `urls`.
- **Connections:**
  - Output → **Split Out - Orphan2**
- **Edge cases:** depends on API response shape containing `urls`.

#### Node: Split Out - Orphan2
- **Type / role:** `Split Out` on `url`.
- **Connections:**
  - Output → **Merge** (as Input 2, index 1)
- **Edge cases:** depends on response shape.

#### Node: Merge
- **Type / role:** `Merge` (combine by fields). Inner-join-like combine on `loc` (input1) and `url` (input2).
- **Config:**
  - Mode: combine, advanced merge by fields: `loc` ↔ `url`
  - `alwaysOutputData: true`
- **Connections:** none further in provided graph.
- **Edge cases:**
  - Requires Input 1 to be the sitemap items (as Sticky Note says). In this workflow JSON, Input 1 is not connected—so this merge won’t produce intended results unless wired.

**Sticky note:**
- **Sticky Note10:** VariationA explanation + Data API doc link https://developer.oncrawl.com/#Data-API and guidance about merge input ordering and variable renaming.

---

### Block 7 — Variation B (Crawl-over-Crawl newly added pages) (present, partial wiring)
**Overview:** Queries Oncrawl crawl-over-crawl API with an OQL to identify newly indexable/canonical pages, splits nested arrays, then merges with sitemap items to recover `lastmod`. Not connected to main path.

**Nodes involved:**
- Get Crawl over crawl
- Split Out - Coc1
- Merge1
- Sticky Note11

#### Node: Get Crawl over crawl
- **Type / role:** `HTTP Request` to Oncrawl crawl_over_crawl endpoint.
- **Config:**
  - URL: `https://app.oncrawl.com/api/v2/data/crawl_over_crawl/69363a80468fbec0e7fe43ca/pages`
  - Method: POST
  - Body: OQL selecting URLs that became indexable/canonical/OK in crawl2 vs crawl1 states.
  - Pagination: `limit: 1000`, `offset: 1`
- **Connections:**
  - Output → **Split Out - Coc1**
- **Edge cases:**
  - Hard-coded crawl_over_crawl id; must be parameterized for real use.
  - Requires Oncrawl API auth and proper plan access.

#### Node: Split Out - Coc1
- **Type / role:** `Split Out` on `urls`.
- **Connections:**
  - Output → **Merge1** (as Input 2, index 1)
- **Edge cases:** depends on response shape.

#### Node: Merge1
- **Type / role:** `Merge` combine by fields `loc` ↔ `url` (same as Variation A).
- **Connections:** none further.
- **Edge cases:** same as Merge—needs sitemap items as Input 1.

**Sticky note:**
- **Sticky Note11:** VariationB explanation + doc link https://developer.oncrawl.com/#Data-API and guidance.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Entry point receiving Oncrawl callback | — | Post - Get Sitemaps | ## Webhook; * When crawl is over, a HTTP callback response is made to the N8N webhook URL.; * doc: https://developer.oncrawl.com/#notification |
| Post - Get Sitemaps | HTTP Request | Ask Oncrawl to discover sitemaps from robots.txt | Webhook | Config | ### Call to Discover_sitemaps endpoint; This endpoint checks the Sitemaps declared in your Robots.txt file.; - doc: https://developer.oncrawl.com/ |
| Config | Set | Central runtime configuration (toggles/keys/URLs) | Post - Get Sitemaps | Get sitemap.xml | ### Config File; **You MUST** update these variables: * SITE_URL * SITEMAP_URL ... * INDEXNOW_KEY ... https://www.bing.com/indexnow/getstarted * INDEXNOW_KEY_URL ...; Variables you can update ... |
| Get sitemap.xml | HTTP Request | Fetch sitemap index XML | Config | Convert sitemap to JSON | ### Parsing the sitemap and generating a list of the urls order by last modification date.; * No need to edit anything here |
| Convert sitemap to JSON | XML | Convert sitemap index XML to JSON | Get sitemap.xml | Get content-specific sitemaps | ### Parsing the sitemap and generating a list of the urls order by last modification date.; * No need to edit anything here |
| Get content-specific sitemaps | Split Out | Iterate sitemapindex.sitemap entries | Convert sitemap to JSON | Get content of each sitemap | ### Parsing the sitemap and generating a list of the urls order by last modification date.; * No need to edit anything here |
| Get content of each sitemap | HTTP Request | Fetch each child sitemap XML | Get content-specific sitemaps | convert page data to JSON | ### Parsing the sitemap and generating a list of the urls order by last modification date.; * No need to edit anything here |
| convert page data to JSON | XML | Convert URL sitemap XML to JSON | Get content of each sitemap | Force urlset.url to array | ### Parsing the sitemap and generating a list of the urls order by last modification date.; * No need to edit anything here |
| Force urlset.url to array | Set | Normalize urlset.url to array | convert page data to JSON | Split Out | ### Parsing the sitemap and generating a list of the urls order by last modification date.; * No need to edit anything here |
| Split Out | Split Out | Split into individual URL items | Force urlset.url to array | Sort | ### Parsing the sitemap and generating a list of the urls order by last modification date.; * No need to edit anything here |
| Sort | Sort | Sort by lastmod desc | Split Out | Assign mandatory sitemap fields | ### Parsing the sitemap and generating a list of the urls order by last modification date.; * No need to edit anything here |
| Assign mandatory sitemap fields | Set | Ensure loc/lastmod are top-level fields | Sort | Filter: lastmod within DAYS_BACK |  |
| Filter: lastmod within DAYS_BACK | Code | Keep only URLs modified within N days | Assign mandatory sitemap fields | Gate: IndexNow; Gate: Google | ### Filter to prevent API rate limiting issues; Cut-off calculation based on Now date -  Date set as DAYS_BACK |
| Gate: IndexNow | Code | Feature flag gate for IndexNow branch | Filter: lastmod within DAYS_BACK | Split In Batches (IndexNow ≤500)1 | ### IndexNow Autosubmitting; * Gate: IndexNow: Is USE_INDEXNOW is true from Config? * Split in Batches... * Build IndexNow payload... * IndexNow Submit... |
| Split In Batches (IndexNow ≤500)1 | Split In Batches | Batch URLs for IndexNow (≤500) | Gate: IndexNow; Wait (IndexNow jitter)1 | Build IndexNow payload | ### IndexNow Autosubmitting; * Gate: IndexNow... |
| Build IndexNow payload | Code | Construct IndexNow JSON payload with host/key/urlList | Split In Batches (IndexNow ≤500)1 | IndexNow Submit | ### IndexNow Autosubmitting; * Gate: IndexNow... |
| IndexNow Submit | HTTP Request | POST payload to IndexNow | Build IndexNow payload | Wait (IndexNow jitter)1 | ### IndexNow Autosubmitting; * Gate: IndexNow... |
| Wait (IndexNow jitter)1 | Wait | Jitter delay between IndexNow batches | IndexNow Submit | Split In Batches (IndexNow ≤500)1 | ### IndexNow Autosubmitting; * Gate: IndexNow... |
| Gate: Google | Code | Feature flag gate for Google branch | Filter: lastmod within DAYS_BACK | Loop Over Items | ## Submit URLs to Google for indexing; (details about gate/loop/check/switch/wait) |
| Loop Over Items | Split In Batches | Iterate URLs (intended 1-by-1) | Gate: Google; Wait2 | Check status | ## Submit URLs to Google for indexing; (details about gate/loop/check/switch/wait) |
| Check status | HTTP Request | Google Search Console URL Inspection | Loop Over Items | Switch | ## Submit URLs to Google for indexing; (details about gate/loop/check/switch/wait) |
| Switch | Switch | Route by coverageState | Check status | is new?; URL Updated | ## Submit URLs to Google for indexing; (details about gate/loop/check/switch/wait) |
| is new? | IF | If sitemap lastmod is after last crawl | Switch | URL Updated; Wait2 | ## Submit URLs to Google for indexing; (details about gate/loop/check/switch/wait) |
| URL Updated | HTTP Request | Google Indexing API publish (URL_UPDATED) | is new?; Switch | Wait2 | ## Submit URLs to Google for indexing; (details about gate/loop/check/switch/wait) |
| Wait2 | Wait | Random delay between Google calls | URL Updated; is new? | Loop Over Items | ## Submit URLs to Google for indexing; (details about gate/loop/check/switch/wait) |
| Config - Orphan | Set | Variation A config example | — | Get Orphan Pages | ## VariationA: Index orphan pages; * API documentation: https://developer.oncrawl.com/#Data-API ... |
| Get Orphan Pages | HTTP Request | Variation A: fetch orphan pages via Oncrawl Data API | Config - Orphan | Split Out - Orphan1 | ## VariationA: Index orphan pages; * API documentation: https://developer.oncrawl.com/#Data-API ... |
| Split Out - Orphan1 | Split Out | Variation A: split `urls` | Get Orphan Pages | Split Out - Orphan2 | ## VariationA: Index orphan pages; * API documentation: https://developer.oncrawl.com/#Data-API ... |
| Split Out - Orphan2 | Split Out | Variation A: split `url` | Split Out - Orphan1 | Merge | ## VariationA: Index orphan pages; * API documentation: https://developer.oncrawl.com/#Data-API ... |
| Merge | Merge | Variation A: merge sitemap items with orphan URLs by loc/url | Split Out - Orphan2 (Input2); (Input1 missing) | — | ## VariationA: Index orphan pages; * API documentation: https://developer.oncrawl.com/#Data-API ... |
| Get Crawl over crawl | HTTP Request | Variation B: fetch crawl-over-crawl delta pages | — | Split Out - Coc1 | ## VariationB: Index newly added pages between a Crawl 1 & a Crawl2; * API documentation: https://developer.oncrawl.com/#Data-API ... |
| Split Out - Coc1 | Split Out | Variation B: split `urls` | Get Crawl over crawl | Merge1 | ## VariationB: Index newly added pages between a Crawl 1 & a Crawl2; * API documentation: https://developer.oncrawl.com/#Data-API ... |
| Merge1 | Merge | Variation B: merge sitemap items with delta URLs by loc/url | Split Out - Coc1 (Input2); (Input1 missing) | — | ## VariationB: Index newly added pages between a Crawl 1 & a Crawl2; * API documentation: https://developer.oncrawl.com/#Data-API ... |
| Sticky Note2 | Sticky Note | Documentation / context | — | — | (content is the note itself) |
| Sticky Note3 | Sticky Note | Documentation | — | — | (content is the note itself) |
| Sticky Note | Sticky Note | Documentation | — | — | (content is the note itself) |
| Sticky Note7 | Sticky Note | Documentation | — | — | (content is the note itself) |
| Sticky Note9 | Sticky Note | Documentation | — | — | (content is the note itself) |
| Sticky Note1 | Sticky Note | Documentation | — | — | (content is the note itself) |
| Sticky Note8 | Sticky Note | Documentation | — | — | (content is the note itself) |
| Sticky Note12 | Sticky Note | Documentation | — | — | (content is the note itself) |
| Sticky Note10 | Sticky Note | Documentation | — | — | (content is the note itself) |
| Sticky Note11 | Sticky Note | Documentation | — | — | (content is the note itself) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: `🏁 Submit URLs for IndexNow & Google Indexing API`
- (Optional) Add tag: `🧠 Indexation`

2) **Add the entry Webhook node**
- Node type: **Webhook**
- Method: **POST**
- Path: generate a unique path (or reuse the provided one)
- Response mode: choose **On Received** (simplest) or keep **Response Node** but then you must add a **Respond to Webhook** node later.
- Expected payload: Oncrawl notification containing `body.project.id`, `body.crawl.id`, etc.

3) **Add Oncrawl “discover_sitemaps” HTTP Request**
- Node type: **HTTP Request**
- Method: **POST**
- URL: `https://app.oncrawl.com/api/v2/projects/{{$json.body.project.id}}/crawl_configs/discover_sitemaps`
- Body type: **JSON**
- Add header: `Content-Type: application/json` (no leading space)
- Paste/configure the `crawl_config` JSON body (adapt placeholders: start_url, notifications, etc.).
- **Authentication (required):**
  - Configure Oncrawl API token (commonly via Authorization header Bearer token, or n8n credentials if you have a custom credential). The JSON does not show it, but Oncrawl requires auth.
- Connect: **Webhook → Post - Get Sitemaps**

4) **Add Config (Set node)**
- Node type: **Set**
- Add fields (minimum required):
  - `SITE_URL` (e.g., `https://www.example.com`)
  - `SITEMAP_URL` = `{{$json.results[0].url}}` (from Oncrawl discover)
  - `DAYS_BACK` (default `7`)
  - `BATCH_SIZE` (default `500`)
  - `USE_GOOGLE` (true/false)
  - `USE_INDEXNOW` (true/false)
  - `INDEXNOW_KEY` (string)
  - `INDEXNOW_KEY_URL` (string, e.g. `https://www.example.com/<key>`)
  - (Optional) `body.crawl.id` = `{{$('Webhook').item.json.body.crawl.id}}`
- Connect: **Post - Get Sitemaps → Config**

5) **Build the sitemap parsing chain**
- **HTTP Request** “Get sitemap.xml”
  - URL: `{{$json.SITEMAP_URL}}`
- **XML** “Convert sitemap to JSON” (default options)
- **Split Out** “Get content-specific sitemaps”
  - Field: `sitemapindex.sitemap`
- **HTTP Request** “Get content of each sitemap”
  - URL: `{{$json.loc}}`
- **XML** “convert page data to JSON”
  - Option: `explicitArray = false`
- **Set** “Force urlset.url to array”
  - Field `urlset.url` (Array) with expression:
    - `{{$json.urlset && $json.urlset.url ? ($json.urlset.url[0] ? $json.urlset.url : [$json.urlset.url]) : []}}`
- **Split Out** “Split Out”
  - Field: `urlset.url`
- **Sort** “Sort”
  - Sort by field `lastmod` descending
- **Set** “Assign mandatory sitemap fields”
  - `loc = {{$json.loc}}`
  - `lastmod = {{$json.lastmod}}`
- Connect in order:
  - **Config → Get sitemap.xml → Convert sitemap to JSON → Get content-specific sitemaps → Get content of each sitemap → convert page data to JSON → Force urlset.url to array → Split Out → Sort → Assign mandatory sitemap fields**

6) **Add freshness filter (Code node)**
- Node type: **Code**
- Paste logic equivalent to:
  - read `DAYS_BACK` from `$items('Config')[0].json.DAYS_BACK || 7`
  - compute cutoff
  - keep items where `lastmod` exists and is within cutoff
  - output `{loc,lastmod}`
- Connect: **Assign mandatory sitemap fields → Filter: lastmod within DAYS_BACK**

7) **IndexNow branch**
- Add **Code** “Gate: IndexNow”
  - Pass items only if `Config.USE_INDEXNOW` is true (boolean or “true”).
- Add **Split In Batches** “Split In Batches (IndexNow ≤500)”
  - Batch size expression: `{{ Math.min(($items('Config')[0].json.BATCH_SIZE || 500), 500) }}`
- Add **Code** “Build IndexNow payload”
  - Build payload: `{host, key, keyLocation, urlList}`
  - Derive `host` from `Config.SITE_URL`
  - Filter URLs by host and dedupe
- Add **HTTP Request** “IndexNow Submit”
  - POST `https://api.indexnow.org/indexnow`
  - JSON body: `{{$json}}`
- Add **Wait** “Wait (IndexNow jitter)”
  - Seconds: `{{ (0.25 + Math.random()*0.75).toFixed(2) }}`
- Connect:
  - **Filter → Gate: IndexNow → Split In Batches**
  - **Split In Batches (output for each batch) → Build IndexNow payload → IndexNow Submit → Wait → back to Split In Batches** (loop)

8) **Google branch**
- Add **Code** “Gate: Google”
  - Pass items only if `Config.USE_GOOGLE` is true.
- Add **Split In Batches** “Loop Over Items”
  - Set batch size to **1** explicitly to ensure one-by-one processing.
- Add **HTTP Request** “Check status” (URL Inspection)
  - POST `https://searchconsole.googleapis.com/v1/urlInspection/index:inspect`
  - Body type: form-urlencoded
  - Params:
    - `inspectionUrl = {{$json.loc}}`
    - `siteUrl = <your Search Console property URL>` (recommended: use an expression from Config, e.g. `{{$items('Config')[0].json.SITE_URL}}`, but ensure it matches the verified property exactly)
  - Enable **full response** if you want status code/body/headers.
  - **Credentials:** Google OAuth2 with Search Console scope (and access to that property).
- Add **Switch**
  - Evaluate: `{{$json.body.inspectionResult.indexStatusResult.coverageState}}`
  - Case 1: equals `Submitted and indexed` → go to IF “is new?”
  - Case 2: equals `Crawled - currently not indexed` → go to “URL Updated”
- Add **IF** “is new?”
  - Condition: `{{$('Loop Over Items').item.json.lastmod}}` is after `{{$json.body.inspectionResult.indexStatusResult.lastCrawlTime}}`
- Add **HTTP Request** “URL Updated” (Indexing API)
  - POST `https://indexing.googleapis.com/v3/urlNotifications:publish`
  - Body:
    - `url = {{$('Loop Over Items').item.json.loc}}`
    - `type = URL_UPDATED`
  - **Credentials:** Google OAuth2 or service account authorized for Indexing API; API enabled in GCP.
- Add **Wait** “Wait2”
  - Seconds: `{{ Math.min(1.5,0.3+3*Math.random()).toFixed(2) }}`
- Connect:
  - **Filter → Gate: Google → Loop Over Items → Check status → Switch**
  - Switch “Submitted and indexed” → **is new?**
    - True → **URL Updated**
    - False → **Wait2**
  - Switch “Crawled - currently not indexed” → **URL Updated**
  - **URL Updated → Wait2 → back to Loop Over Items** (loop)

9) **(Optional) Add Variation A wiring**
- Use nodes: **Get Orphan Pages + Split Outs + Merge**
- Connect **Assign mandatory sitemap fields** (or filtered list) to **Merge input 1**
- Connect orphan URL stream to **Merge input 2**
- Then route merged items into the same Filter/Gates or directly to IndexNow/Google paths depending on your intent.

10) **(Optional) Add Variation B wiring**
- Parameterize the crawl_over_crawl id instead of hardcoding.
- Connect sitemap items to **Merge1 input 1**, delta URL stream to **Merge1 input 2**.
- Continue into Filter/Gates.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Oncrawl API documentation | https://developer.oncrawl.com/ |
| Oncrawl Notification (webhook callback) documentation | https://developer.oncrawl.com/#notification |
| Oncrawl Data API documentation (used in variations) | https://developer.oncrawl.com/#Data-API |
| IndexNow key creation | https://www.bing.com/indexnow/getstarted |
| Reference workflow page mentioned in sticky note | https://n8n.io/workflows/8778-workflow-for-submitting-changed-sitemap-urls-using-google-indexing-api-and-bing-indexnow/ |
| Important operational note | `Check status` node currently has an empty `siteUrl` parameter; it must match a verified Search Console property or Google will return an error. |
| Structural note | Webhook is configured with `responseMode: responseNode` but there is no “Respond to Webhook” node in this JSON; add one or change response mode to avoid hanging executions. |