Filter sitemap URLs and inspect indexing status with Google Search Console

https://n8nworkflows.xyz/workflows/filter-sitemap-urls-and-inspect-indexing-status-with-google-search-console-14991


# Filter sitemap URLs and inspect indexing status with Google Search Console

Let me analyze this workflow carefully and create a comprehensive reference document.

The workflow is about filtering sitemap URLs and inspecting their indexing status with Google Search Console, then publishing (requesting indexing) for URLs that are recently modified and have a NEUTRAL verdict (not yet indexed).

Let me break down the nodes and their connections:

1. **Trigger Daily** (Schedule Trigger) → **Configuration**
2. **Configuration** (Set) → **Read Sitemap**
3. **Read Sitemap** (HTTP Request) → **Parse XML**
4. **Parse XML** (XML) → **Split Out URLs**
5. **Split Out URLs** (Split Out) → **Filter Last Modified Pages**
6. **Filter Last Modified Pages** (Filter) → **Inspect URLs**
7. **Inspect URLs** (HTTP Request) → **Filter NEUTRAL Verdict**
8. **Filter NEUTRAL Verdict** (Filter) → **Publish URLs**
9. **Publish URLs** (HTTP Request)

Sticky Notes:
- Sticky Note: Covers Trigger Daily and Configuration - "### 0. Setup & Prerequisites..."
- Sticky Note1: Covers Read Sitemap, Parse XML, Split Out URLs - "### 1. Sitemap Processing..."
- Sticky Note2: Covers Filter Last Modified Pages, Inspect URLs, Filter NEUTRAL Verdict - "### 2. Efficiency Logic..."
- Sticky Note3: Covers Publish URLs - "### 3. Google Indexing API..."

Let me now create the full document. 1. Workflow Overview

This workflow automates selective Google indexing for recently-updated web pages. It runs on a daily schedule, fetches a website's XML sitemap, and applies two filters to minimize API usage: first it keeps only pages modified within the last 7 days, then it checks each remaining URL against the Google Search Console URL Inspection API and retains only those with a **NEUTRAL** verdict (i.e., not yet indexed or pending). The final filtered list is submitted to the Google Indexing API as `URL_UPDATED` notifications, signaling Google to prioritize crawling.

**Logical blocks:**

| Block | Name | Nodes | Purpose |
|-------|------|-------|---------|
| 0 | Setup & Prerequisites | Trigger Daily, Configuration | Schedule the run and provide configurable parameters (sitemap URL, modification window) |
| 1 | Sitemap Processing | Read Sitemap, Parse XML, Split Out URLs | Fetch, parse, and individualize every URL entry from the XML sitemap |
| 2 | Efficiency Logic | Filter Last Modified Pages, Inspect URLs, Filter NEUTRAL Verdict | Narrow down candidates by recency then by indexing status to save API quota |
| 3 | Google Indexing API | Publish URLs | Request Google to crawl the qualifying URLs |

---

## 2. Block-by-Block Analysis

### Block 0 — Setup & Prerequisites

**Overview:** A daily schedule trigger fires the workflow. A Set node centralizes configuration so the sitemap URL and recency threshold are easy to change without touching downstream nodes.

**Nodes Involved:** Trigger Daily, Configuration

---

#### Trigger Daily

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.scheduleTrigger` (v1.3) |
| **Technical Role** | Entry point; fires the workflow on a recurring schedule |
| **Configuration** | Interval set to default (every 1 day). No custom cron expression, so it runs once per day. |
| **Key Expressions** | None |
| **Input** | None (root trigger) |
| **Output** | Sends empty item to **Configuration** |
| **Edge Cases / Failures** | If the n8n instance is down at the scheduled time, the execution is skipped (no retry unless manually triggered). Timezone depends on the n8n server's TZ setting. |

---

#### Configuration

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Defines two runtime parameters consumed by downstream nodes |
| **Configuration** | Two string assignments: |
| | • `sitemap_url` = `https://example.com/sitemap.xml` (hard-coded default; user should replace) |
| | • `modified_after` = `={{ $now.minus(1, 'week').toISO() }}` — ISO 8601 timestamp of 7 days ago |
| **Key Expressions** | `$now.minus(1, 'week').toISO()` — dynamically computes the cutoff timestamp at each run |
| **Input** | From **Trigger Daily** |
| **Output** | To **Read Sitemap**; also referenced by name in **Inspect URLs** and **Filter Last Modified Pages** |
| **Edge Cases / Failures** | If `sitemap_url` is invalid or unreachable, the downstream HTTP Request will fail. If the expression engine cannot resolve `$now` (e.g., extremely old n8n version), `modified_after` will be null and the filter will pass no items. |

**Sticky Note covering this block:**
> ### 0. Setup & Prerequisites
> This workflow selectively indexes pages by filtering for recent updates and checking live status.
>
> **Preparation:**
> 1. Enable Search Console & Indexing APIs in Google Cloud.
> 2. Add Service Account JSON to n8n credentials.
> 3. Verify Service Account email as 'Owner' in GSC.
> 4. Set your sitemap URL in the 'Configuration' node.

---

### Block 1 — Sitemap Processing

**Overview:** This block fetches the remote XML sitemap, converts it to a JSON structure, then splits the array of `<url>` entries into individual items so that each URL can be processed independently downstream.

**Nodes Involved:** Read Sitemap, Parse XML, Split Out URLs

---

#### Read Sitemap

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.httpRequest` (v4.3) |
| **Technical Role** | HTTP GET to retrieve the raw XML sitemap |
| **Configuration** | URL = `={{ $('Configuration').item.json.sitemap_url }}`; Method = GET (default); No auth; No special options |
| **Key Expressions** | `$('Configuration').item.json.sitemap_url` — reads the URL from the Configuration node |
| **Input** | From **Configuration** |
| **Output** | Raw XML string in `data` field → **Parse XML** |
| **Edge Cases / Failures** | • HTTP 404 or 5xx → node error, execution stops. • Very large sitemaps may hit n8n's binary/string payload limit. • If the server returns a non-XML content-type, the XML parser may still work as long as the body is valid XML. • Redirects are followed by default. |

---

#### Parse XML

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.xml` (v1) |
| **Technical Role** | Converts the XML sitemap string into a JSON object |
| **Configuration** | No options customized (default parsing behavior) |
| **Key Expressions** | None |
| **Input** | From **Read Sitemap** (XML string) |
| **Output** | JSON object containing a `urlset.url` array → **Split Out URLs** |
| **Edge Cases / Failures** | • Malformed XML will cause a parse error. • Very deep or extremely large structures may produce memory issues. • Namespaces in the sitemap XML may add prefix annotations in the resulting JSON keys. |

---

#### Split Out URLs

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.splitOut` (v1) |
| **Technical Role** | Explodes the `urlset.url` array into separate items, one per `<url>` entry |
| **Configuration** | `fieldToSplitOut` = `urlset.url`; no additional options |
| **Key Expressions** | None |
| **Input** | From **Parse XML** (single item with array) |
| **Output** | Multiple items, each with one URL object containing fields like `loc` (URL) and optionally `lastmod` → **Filter Last Modified Pages** |
| **Edge Cases / Failures** | • If `urlset.url` is missing or not an array (e.g., empty sitemap), the node outputs zero items and the workflow effectively ends. • If individual URL entries lack a `loc` field, downstream expressions referencing `$json.loc` will resolve to `undefined`. |

**Sticky Note covering this block:**
> ### 1. Sitemap Processing
> Fetches the XML sitemap, parses it to JSON, and splits it into individual URL objects for item-by-item processing.

---

### Block 2 — Efficiency Logic

**Overview:** Two sequential filters dramatically reduce the number of URLs that hit Google APIs. First, a date filter keeps only pages modified after the 7-day cutoff. Then each surviving URL is inspected live via the Google Search Console API; only those returning a NEUTRAL verdict (not indexed yet) proceed to indexing.

**Nodes Involved:** Filter Last Modified Pages, Inspect URLs, Filter NEUTRAL Verdict

---

#### Filter Last Modified Pages

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.filter` (v2.3) |
| **Technical Role** | Keeps only URLs whose `lastmod` date is after the computed `modified_after` timestamp |
| **Configuration** | Single condition (combinator = `and`): |
| | • Operator: `dateTime` → `after` |
| | • Left value: `={{ $json.lastmod }}` |
| | • Right value: `={{ $('Configuration').item.json.modified_after }}` |
| | Type validation: strict; case sensitive: true |
| **Key Expressions** | `$json.lastmod` — the `<lastmod>` value from the sitemap entry; `$('Configuration').item.json.modified_after` — the 7-days-ago ISO timestamp |
| **Input** | From **Split Out URLs** |
| **Output** | Only recently-modified items → **Inspect URLs** |
| **Edge Cases / Failures** | • If `lastmod` is missing or not a valid ISO date string, strict type validation will likely fail and the item is filtered out. • Sitemaps that omit `<lastmod>` entirely will yield zero items, stopping the workflow. • Timezone mismatches between `lastmod` (often UTC) and `$now` (server TZ) could cause off-by-one-day discrepancies. |

---

#### Inspect URLs

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.httpRequest` (v4.3) |
| **Technical Role** | Calls the Google Search Console URL Inspection API to check the live indexing status of each URL |
| **Configuration** | |
| | • URL: `https://searchconsole.googleapis.com/v1/urlInspection/index:inspect` |
| | • Method: POST |
| | • Authentication: `predefinedCredentialType` → `googleApi` (Google Service Account) |
| | • Body (JSON): `inspectionUrl` = `={{ $json.loc }}`, `siteUrl` = `=sc-domain:{{ $('Configuration').item.json.sitemap_url.extractDomain() }}` |
| | • Batching: batch size = 1 (processes one request at a time) |
| **Key Expressions** | `$json.loc` — the URL to inspect; `$('Configuration').item.json.sitemap_url.extractDomain()` — extracts the domain from the configured sitemap URL and prefixes with `sc-domain:` (the GSC domain property format) |
| **Input** | From **Filter Last Modified Pages** |
| **Output** | Inspection result object (including `inspectionResult.indexStatusResult.verdict`) → **Filter NEUTRAL Verdict** |
| **Edge Cases / Failures** | • 401/403 if the Service Account lacks access to the GSC property. • 429 rate limiting if too many requests are sent rapidly (batching = 1 mitigates this but doesn't add delay). • `extractDomain()` is a custom n8n string helper; if the sitemap URL is malformed, this will fail. • The API may return `VERDICT_UNSPECIFIED` or other verdict strings if the URL is newly discovered; only `NEUTRAL` passes downstream. • Network timeouts on large sitemaps could cause partial execution. |
| **Credential** | Google Service Account (JSON key). Must be granted Owner role on the GSC property. |

---

#### Filter NEUTRAL Verdict

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.filter` (v2.3) |
| **Technical Role** | Passes through only URLs whose indexing verdict is `NEUTRAL` (not yet indexed or pending) |
| **Configuration** | Single condition (combinator = `and`): |
| | • Operator: `string` → `equals` |
| | • Left value: `={{ $json.inspectionResult.indexStatusResult.verdict }}` |
| | • Right value: `NEUTRAL` |
| | Type validation: strict; case sensitive: true |
| **Key Expressions** | `$json.inspectionResult.indexStatusResult.verdict` — the verdict field returned by the Inspection API |
| **Input** | From **Inspect URLs** |
| **Output** | Only NEUTRAL-verdict items → **Publish URLs** |
| **Edge Cases / Failures** | • If the Inspection API response structure changes (e.g., nested differently), the expression will resolve to `undefined` and no items pass. • Case sensitivity means `neutral` (lowercase) would be rejected — this matches Google's documented uppercase verdicts. • If a URL has verdict `PASS` (already indexed), it is intentionally excluded to save API quota. |

**Sticky Note covering this block:**
> ### 2. Efficiency Logic
> **Step A:** Filters for pages modified in the last 7 days.
> **Step B:** Inspects current status via GSC API. Only 'NEUTRAL' (not yet indexed or pending) pages proceed to save API quota.

---

### Block 3 — Google Indexing API

**Overview:** The final step sends each qualifying URL to Google's Indexing API as a `URL_UPDATED` notification, requesting priority crawling. This is a signal, not a guarantee of immediate indexing.

**Nodes Involved:** Publish URLs

---

#### Publish URLs

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.httpRequest` (v4.3) |
| **Technical Role** | Submits an indexing notification for each filtered URL via the Google Indexing API |
| **Configuration** | |
| | • URL: `https://indexing.googleapis.com/v3/urlNotifications:publish` |
| | • Method: POST |
| | • Authentication: `predefinedCredentialType` → `googleApi` (same Service Account) |
| | • Body (JSON): `url` = `={{ $('Split Out URLs').item.json.loc }}`, `type` = `URL_UPDATED` |
| | • Batching: batch size = 1 |
| **Key Expressions** | `$('Split Out URLs').item.json.loc` — retrieves the original URL from the split-out data (not from the inspection response, which may not contain `loc` at the same path) |
| **Input** | From **Filter NEUTRAL Verdict** |
| **Output** | None (terminal node; API response is logged but not further processed) |
| **Edge Cases / Failures** | • 403 if the Service Account is not authorized for the Indexing API. • 429 rate limiting — Google allows a limited number of requests per day; large sitemaps may exceed this. • The `URL_UPDATED` type is appropriate for re-indexing modified pages; `URL_DELETED` would be used for removal. • If `$('Split Out URLs').item.json.loc` resolves to `undefined` (e.g., item reference is stale after multiple splits/filters), the API call will fail with an invalid URL. |
| **Credential** | Same Google Service Account credential as Inspect URLs. The Indexing API must be enabled in Google Cloud and the Service Account must be an Owner on the GSC property. |

**Sticky Note covering this block:**
> ### 3. Google Indexing API
> Sends the final list of filtered URLs to Google for priority crawling.
> *Note: This signals Google to crawl, but doesn't guarantee instant indexing.*

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger Daily | scheduleTrigger (v1.3) | Daily schedule entry point | — | Configuration | ### 0. Setup & Prerequisites — This workflow selectively indexes pages by filtering for recent updates and checking live status. **Preparation:** 1. Enable Search Console & Indexing APIs in Google Cloud. 2. Add Service Account JSON to n8n credentials. 3. Verify Service Account email as 'Owner' in GSC. 4. Set your sitemap URL in the 'Configuration' node. |
| Configuration | set (v3.4) | Centralized parameters (sitemap URL, recency window) | Trigger Daily | Read Sitemap | ### 0. Setup & Prerequisites — This workflow selectively indexes pages by filtering for recent updates and checking live status. **Preparation:** 1. Enable Search Console & Indexing APIs in Google Cloud. 2. Add Service Account JSON to n8n credentials. 3. Verify Service Account email as 'Owner' in GSC. 4. Set your sitemap URL in the 'Configuration' node. |
| Read Sitemap | httpRequest (v4.3) | HTTP GET to fetch the XML sitemap | Configuration | Parse XML | ### 1. Sitemap Processing — Fetches the XML sitemap, parses it to JSON, and splits it into individual URL objects for item-by-item processing. |
| Parse XML | xml (v1) | Convert XML sitemap to JSON | Read Sitemap | Split Out URLs | ### 1. Sitemap Processing — Fetches the XML sitemap, parses it to JSON, and splits it into individual URL objects for item-by-item processing. |
| Split Out URLs | splitOut (v1) | Explode URL array into individual items | Parse XML | Filter Last Modified Pages | ### 1. Sitemap Processing — Fetches the XML sitemap, parses it to JSON, and splits it into individual URL objects for item-by-item processing. |
| Filter Last Modified Pages | filter (v2.3) | Keep only pages modified within the last 7 days | Split Out URLs | Inspect URLs | ### 2. Efficiency Logic — **Step A:** Filters for pages modified in the last 7 days. **Step B:** Inspects current status via GSC API. Only 'NEUTRAL' (not yet indexed or pending) pages proceed to save API quota. |
| Inspect URLs | httpRequest (v4.3) | Call GSC URL Inspection API per URL | Filter Last Modified Pages | Filter NEUTRAL Verdict | ### 2. Efficiency Logic — **Step A:** Filters for pages modified in the last 7 days. **Step B:** Inspects current status via GSC API. Only 'NEUTRAL' (not yet indexed or pending) pages proceed to save API quota. |
| Filter NEUTRAL Verdict | filter (v2.3) | Keep only NEUTRAL (not-indexed) verdicts | Inspect URLs | Publish URLs | ### 2. Efficiency Logic — **Step A:** Filters for pages modified in the last 7 days. **Step B:** Inspects current status via GSC API. Only 'NEUTRAL' (not yet indexed or pending) pages proceed to save API quota. |
| Publish URLs | httpRequest (v4.3) | Submit URL_UPDATED notification to Google Indexing API | Filter NEUTRAL Verdict | — (terminal) | ### 3. Google Indexing API — Sends the final list of filtered URLs to Google for priority crawling. *Note: This signals Google to crawl, but doesn't guarantee instant indexing.* |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set its name to `Smart google indexing: sitemap filter and url inspection`.

2. **Add a Schedule Trigger node.**
   - Type: `Schedule Trigger`
   - Version: 1.3
   - Rule: Interval, set to run every 1 day (default).
   - Position it at the far left of the canvas.

3. **Add a Set node** named `Configuration`.
   - Type: `Set` (v3.4)
   - Assignments:
     - `sitemap_url` → type String → value `https://example.com/sitemap.xml` (replace with your actual sitemap URL).
     - `modified_after` → type String → value `={{ $now.minus(1, 'week').toISO() }}`.
   - Connect **Trigger Daily** → **Configuration**.

4. **Add an HTTP Request node** named `Read Sitemap`.
   - Type: `HTTP Request` (v4.3)
   - Method: GET
   - URL: `={{ $('Configuration').item.json.sitemap_url }}`
   - Authentication: None
   - Options: default
   - Connect **Configuration** → **Read Sitemap**.

5. **Add an XML node** named `Parse XML`.
   - Type: `XML` (v1)
   - Options: default (no custom settings)
   - Connect **Read Sitemap** → **Parse XML**.

6. **Add a Split Out node** named `Split Out URLs`.
   - Type: `Split Out` (v1)
   - Field to Split Out: `urlset.url`
   - Options: default
   - Connect **Parse XML** → **Split Out URLs**.

7. **Add a Filter node** named `Filter Last Modified Pages`.
   - Type: `Filter` (v2.3)
   - Combinator: AND
   - Condition:
     - Operator: `DateTime` → `after`
     - Left value: `={{ $json.lastmod }}`
     - Right value: `={{ $('Configuration').item.json.modified_after }}`
   - Type validation: strict; case sensitive: true
   - Connect **Split Out URLs** → **Filter Last Modified Pages**.

8. **Configure Google Service Account credential in n8n.**
   - In n8n credentials, create a new `Google API` credential.
   - Upload the Service Account JSON key file.
   - Note the credential name (e.g., "Google Service Account account") — you will reference it in the next two HTTP Request nodes.

9. **Add an HTTP Request node** named `Inspect URLs`.
   - Type: `HTTP Request` (v4.3)
   - Method: POST
   - URL: `https://searchconsole.googleapis.com/v1/urlInspection/index:inspect`
   - Authentication: Predefined Credential Type → `Google API`
   - Credential: select the Service Account credential created in step 8
   - Send Body: enabled
   - Body Parameters:
     - `inspectionUrl` → `={{ $json.loc }}`
     - `siteUrl` → `=sc-domain:{{ $('Configuration').item.json.sitemap_url.extractDomain() }}`
   - Options → Batching → Batch Size: 1
   - Connect **Filter Last Modified Pages** → **Inspect URLs**.

10. **Add a Filter node** named `Filter NEUTRAL Verdict`.
    - Type: `Filter` (v2.3)
    - Combinator: AND
    - Condition:
      - Operator: `String` → `equals`
      - Left value: `={{ $json.inspectionResult.indexStatusResult.verdict }}`
      - Right value: `NEUTRAL`
    - Type validation: strict; case sensitive: true
    - Connect **Inspect URLs** → **Filter NEUTRAL Verdict**.

11. **Add an HTTP Request node** named `Publish URLs`.
    - Type: `HTTP Request` (v4.3)
    - Method: POST
    - URL: `https://indexing.googleapis.com/v3/urlNotifications:publish`
    - Authentication: Predefined Credential Type → `Google API`
    - Credential: same Service Account credential
    - Send Body: enabled
    - Body Parameters:
      - `url` → `={{ $('Split Out URLs').item.json.loc }}`
      - `type` → `URL_UPDATED`
    - Options → Batching → Batch Size: 1
    - Connect **Filter NEUTRAL Verdict** → **Publish URLs**.

12. **Add Sticky Notes** for documentation (optional but recommended):
    - Place a sticky note covering Trigger Daily and Configuration with the "Setup & Prerequisites" content.
    - Place a sticky note covering Read Sitemap through Split Out URLs with the "Sitemap Processing" content.
    - Place a sticky note covering Filter Last Modified Pages through Filter NEUTRAL Verdict with the "Efficiency Logic" content.
    - Place a sticky note covering Publish URLs with the "Google Indexing API" content.

13. **Activate the workflow** by toggling it to Active.

---

### Google Cloud Prerequisites (must be completed before the workflow runs)

| Step | Action |
|------|--------|
| 1 | In Google Cloud Console, enable the **Google Search Console API** (`searchconsole.googleapis.com`) and the **Indexing API** (`indexing.googleapis.com`). |
| 2 | Create a Service Account and generate a JSON key. Download the key file. |
| 3 | In Google Search Console, add the Service Account's email address as an **Owner** (or at least Full User) of the property you want to index. Use the `sc-domain:example.com` format if it's a domain property. |
| 4 | Upload the JSON key into n8n as a `Google API` credential. |
| 5 | Update the `sitemap_url` value in the Configuration node to point to your site's actual sitemap. |

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The `extractDomain()` helper strips the path from a URL, returning only the domain (e.g., `https://example.com/sitemap.xml` → `example.com`). It is a built-in n8n string method available in expressions. | Used in the `siteUrl` body parameter of the Inspect URLs node |
| The `sc-domain:` prefix is required by the GSC URL Inspection API when the property is registered as a Domain property (not a URL-prefix property). If your property is a URL-prefix property, use the full URL instead (e.g., `https://example.com/`). | Google Search Console API documentation |
| Google's Indexing API has a daily quota (typically 200 requests per day for non-production projects). Plan sitemap size and filter aggressiveness accordingly. | https://developers.google.com/search/apis/indexing-api/v3/using-api |
| The `URL_UPDATED` notification type is appropriate for modified pages. Use `URL_DELETED` for pages that have been removed and should be de-indexed. | https://developers.google.com/search/apis/indexing-api/v3/using-api |
| The workflow uses batch size = 1 on both Google API calls to avoid overloading rate limits. For small sitemaps this is fine; for larger ones, consider adding a `Wait` node between Inspect URLs iterations or using a Slack/email notification for 429 errors. | General rate-limiting best practice |
| The `lastmod` field in sitemaps is optional per the sitemap protocol. If your sitemap omits it, the Filter Last Modified Pages node will exclude all URLs. Ensure your CMS or static site generator outputs `<lastmod>` tags. | https://www.sitemaps.org/protocol.html |
| This workflow does not handle pagination in sitemaps (sitemap index files pointing to child sitemaps). If your site uses a sitemap index, you would need an additional loop to fetch each child sitemap before the Parse XML step. | Sitemap index protocol |
| The Inspection API response may include a `verdict` of `VERDICT_UNSPECIFIED` for very new URLs. These will be filtered out by the NEUTRAL filter. You may want to add an OR condition for `VERDICT_UNSPECIFIED` if you want to submit those as well. | Google Search Console URL Inspection API |