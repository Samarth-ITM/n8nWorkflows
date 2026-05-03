Compare website SEO gaps using Google Sheets, HTML and Gemini AI

https://n8nworkflows.xyz/workflows/compare-website-seo-gaps-using-google-sheets--html-and-gemini-ai-14151


# Compare website SEO gaps using Google Sheets, HTML and Gemini AI

# 1. Workflow Overview

This workflow performs an SEO gap comparison between a target website and a competitor website using Google Sheets as both input and storage, HTML scraping for page-level SEO data extraction, and Google Gemini for final comparative analysis.

Its main use case is to process rows in a Google Sheet where each row contains:
- your website URL,
- a competitor website URL,
- a processing status.

For each row marked `NEW`, the workflow:
1. reads the input pair,
2. scrapes homepage SEO data from both websites,
3. discovers important internal pages,
4. scrapes and stores page-level SEO metrics for both sites,
5. merges the collected datasets,
6. sends them to Gemini for gap analysis,
7. stores the structured report in Google Sheets,
8. marks the input row as processed.

## 1.1 Input Reception and Validation
The workflow starts manually, reads rows from the input Google Sheet, and only continues for rows whose `Status` is `NEW`.

## 1.2 Homepage SEO Data Collection
It fetches the homepage HTML for both the target and competitor websites, extracts common SEO elements, computes summary metrics, and stores the first row of data for each site.

## 1.3 Internal Page Discovery
From each homepage, it extracts all links, filters them to “important” URLs based on keyword heuristics such as `about`, `service`, `blog`, and `contact`, then writes those URLs into site-specific Google Sheets.

## 1.4 Page-by-Page Scraping for Your Website
It loops through the discovered URLs for the target website, requests each page, extracts SEO data, computes metrics, and updates the corresponding row in the target sheet.

## 1.5 Page-by-Page Scraping for Competitor Website
It performs the same iterative scraping and updating process for the competitor site.

## 1.6 Dataset Consolidation and AI Analysis
Once both sheets are populated, it retrieves the collected records, merges the datasets, sends them to Gemini with a structured prompt, and parses the model response as JSON.

## 1.7 Reporting and Status Update
The final AI-generated SEO gap report is appended to a report sheet, and the original input row is updated from `NEW` to `DONE` with a processed timestamp.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

**Overview:**  
This block manually starts the workflow, reads input rows from Google Sheets, and filters the process to only entries that are explicitly marked as new. It also normalizes the input field names for downstream use.

**Nodes Involved:**  
- When clicking ‘Execute workflow’
- Get row(s) in sheet
- If
- Edit Fields

### Node Details

#### 1) When clicking ‘Execute workflow’
- **Type and technical role:** `Manual Trigger`; starts the workflow on demand from the n8n editor.
- **Configuration choices:** No parameters configured.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Get row(s) in sheet`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None beyond manual execution context.
- **Sub-workflow reference:** None.

#### 2) Get row(s) in sheet
- **Type and technical role:** `Google Sheets`; reads rows from the input sheet.
- **Configuration choices:**  
  - Spreadsheet: `SEO_Workflow`
  - Sheet: `INPUT_WEBSITES`
  - Operation appears to be row retrieval without additional filters.
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:** Input from `When clicking ‘Execute workflow’`; output to `If`.
- **Version-specific requirements:** Type version `4.7`; requires valid Google Sheets credentials.
- **Edge cases or potential failure types:**  
  - OAuth/authentication issues
  - Missing spreadsheet or sheet access
  - Empty sheet returns no actionable items
  - Unexpected column naming differences
- **Sub-workflow reference:** None.

#### 3) If
- **Type and technical role:** `If`; filters incoming items.
- **Configuration choices:** Checks whether `{{$json.Status}}` equals `NEW`.
- **Key expressions or variables used:** `={{ $json.Status }}`
- **Input and output connections:** Input from `Get row(s) in sheet`; true output goes to `Edit Fields`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Missing `Status` field
  - Case sensitivity means `new`, `New`, or trailing spaces will fail the condition
- **Sub-workflow reference:** None.

#### 4) Edit Fields
- **Type and technical role:** `Set`; maps source column names to simpler internal variable names.
- **Configuration choices:** Creates:
  - `your_site` from `Your_Website`
  - `competitor_site` from `Competitor_Website`
  - `row_number` from `row_number`
- **Key expressions or variables used:**  
  - `={{ $json.Your_Website }}`
  - `={{ $json.Competitor_Website }}`
  - `={{ $json.row_number }}`
- **Input and output connections:** Input from `If`; outputs to both `My Website HTTP` and `HTTP (for Competitor)`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Missing or malformed URLs
  - `row_number` may be absent depending on sheet retrieval behavior
- **Sub-workflow reference:** None.

---

## 2.2 Homepage SEO Data Collection

**Overview:**  
This block requests the homepage for both sites, extracts structural SEO elements from the HTML, computes page-level metrics like word count and link counts, and appends the homepage record into dedicated Google Sheets.

**Nodes Involved:**  
- My Website HTTP
- HTML (for My website)
- My Website Data
- Append First Row of My website
- HTTP (for Competitor)
- HTML (for Competitor Website)
- Competitor Data
- Append in Competitor Sheet

### Node Details

#### 5) My Website HTTP
- **Type and technical role:** `HTTP Request`; downloads the HTML of the target homepage.
- **Configuration choices:**  
  - URL: `={{ $json.your_site }}`
  - Response handling enabled with default response object
- **Key expressions or variables used:** `={{ $json.your_site }}`
- **Input and output connections:** Input from `Edit Fields`; output to `HTML (for My website)`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Invalid URL
  - DNS/SSL errors
  - Redirects or bot protection
  - Timeout or 403/404/5xx responses
- **Sub-workflow reference:** None.

#### 6) HTML (for My website)
- **Type and technical role:** `HTML`; extracts SEO-relevant fields from homepage HTML.
- **Configuration choices:** Extracts:
  - `title` from `<title>`
  - `meta_description` from `meta[name="description"]`
  - `h1`
  - `h2` as array
  - `content` from all `<p>` as array
  - `internal_links` from anchor tags starting with `/`
  - `external_links` from anchor tags starting with `http`
  - `image` from all `<img src>`
  - `all_links` from all anchors
- **Key expressions or variables used:** CSS selectors only.
- **Input and output connections:** Input from `My Website HTTP`; outputs to `My Website Data` and `All Links (My Website)`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Invalid or empty HTML
  - Dynamic/SPAs may return little server-side HTML
  - Field name inconsistency later: this node outputs `image`, but code nodes expect `images`
- **Sub-workflow reference:** None.

#### 7) My Website Data
- **Type and technical role:** `Code`; transforms extracted homepage SEO fields into a structured row.
- **Configuration choices:**  
  - Joins paragraph text and counts words
  - Deduplicates trimmed H2s
  - Counts internal/external links
  - Sets `Website` and `Page_URL` to the homepage URL
  - Adds `Collected_Date`
- **Key expressions or variables used:**  
  - `$('Edit Fields').first().json.your_site`
  - `new Date().toISOString().split('T')[0]`
- **Input and output connections:** Input from `HTML (for My website)`; output to `Append First Row of My website`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - `item.images` is referenced, but extractor key is `image`; image count will likely be `0`
  - If `content` is missing, word count becomes `0`
- **Sub-workflow reference:** None.

#### 8) Append First Row of My website
- **Type and technical role:** `Google Sheets`; appends the homepage SEO summary row to the target-site data sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet: `YOUR_SEO_DATA`
  - Maps fields including `ID` from the original input row
- **Key expressions or variables used:**  
  - `={{ $('Get row(s) in sheet').item.json.ID }}`
  - Field mappings from current JSON
- **Input and output connections:** Input from `My Website Data`; no downstream connection.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Missing `ID`
  - Sheet schema mismatch
  - Append duplicates if rerun without cleanup
- **Sub-workflow reference:** None.

#### 9) HTTP (for Competitor)
- **Type and technical role:** `HTTP Request`; downloads competitor homepage HTML.
- **Configuration choices:**  
  - URL: `={{ $('Edit Fields').item.json.competitor_site }}`
  - Response format explicitly set to text
- **Key expressions or variables used:** `={{ $('Edit Fields').item.json.competitor_site }}`
- **Input and output connections:** Input from `Edit Fields`; output to `HTML (for Competitor Website)`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:** Same as target homepage request.
- **Sub-workflow reference:** None.

#### 10) HTML (for Competitor Website)
- **Type and technical role:** `HTML`; extracts SEO elements from competitor homepage HTML.
- **Configuration choices:** Same extraction pattern as target homepage.
- **Key expressions or variables used:** CSS selectors only.
- **Input and output connections:** Input from `HTTP (for Competitor)`; outputs to `Competitor Data` and `All Links (Competitor Website)`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Dynamic pages may not expose SEO fields in raw HTML
  - Same `image` vs `images` mismatch issue downstream
- **Sub-workflow reference:** None.

#### 11) Competitor Data
- **Type and technical role:** `Code`; builds a structured homepage SEO row for the competitor site.
- **Configuration choices:** Similar to `My Website Data`, with output field `Competitor_Website` later mapped as `Website`.
- **Key expressions or variables used:**  
  - `$('Edit Fields').first().json.competitor_site`
  - `new Date().toISOString().split('T')[0]`
- **Input and output connections:** Input from `HTML (for Competitor Website)`; output to `Append in Competitor Sheet`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Same `images` counting bug due to extractor key mismatch
- **Sub-workflow reference:** None.

#### 12) Append in Competitor Sheet
- **Type and technical role:** `Google Sheets`; appends competitor homepage metrics.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet: `COMPETITOR_SEO_DATA`
  - Includes original input `ID`
- **Key expressions or variables used:**  
  - `={{ $('Get row(s) in sheet').item.json.ID }}`
- **Input and output connections:** Input from `Competitor Data`; no downstream connection.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Duplicate appends
  - Schema mismatch
- **Sub-workflow reference:** None.

---

## 2.3 Internal Page Discovery

**Overview:**  
This block identifies candidate internal pages for deeper SEO analysis. It uses homepage links, filters them with a keyword heuristic, inserts placeholder rows into the relevant sheet, and then re-reads them for iterative processing.

**Nodes Involved:**  
- All Links (My Website)
- Wait3
- Inserted All Links in sheets
- Wait4
- Get Rows from All links
- All Links (Competitor Website)
- Inserted all Links (Competitor Website)
- Wait2
- Get row(s) in sheet2

### Node Details

#### 13) All Links (My Website)
- **Type and technical role:** `Code`; filters homepage links into important URLs.
- **Configuration choices:**  
  - Base URL = normalized `your_site` without trailing slash
  - Includes keywords: `about`, `service`, `solution`, `product`, `blog`, `case`, `portfolio`, `contact`, `career`
  - Converts relative URLs to absolute
  - Removes anchors and query strings
  - Always includes homepage
  - Deduplicates final list
- **Key expressions or variables used:**  
  - `$('Edit Fields').first().json.your_site`
  - `$json.all_links || []`
- **Input and output connections:** Input from `HTML (for My website)`; output to `Wait3`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Only accepts links starting with `/` or the exact base URL
  - Ignores absolute same-domain links on alternate subdomain or trailing slash variations
  - Heuristic may miss important pages not containing listed keywords
- **Sub-workflow reference:** None.

#### 14) Wait3
- **Type and technical role:** `Wait`; introduces delay before sheet append.
- **Configuration choices:** Wait amount `3`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `All Links (My Website)`; output to `Inserted All Links in sheets`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Wait duration unit depends on node defaults; likely seconds but should be verified in UI
- **Sub-workflow reference:** None.

#### 15) Inserted All Links in sheets
- **Type and technical role:** `Google Sheets`; appends discovered target-site URLs as placeholder rows.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet: `YOUR_SEO_DATA`
  - Only `Page_URL` is populated
- **Key expressions or variables used:** `={{ $json.page_url }}`
- **Input and output connections:** Input from `Wait3`; output to `Wait4`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Duplicate Page_URL rows if workflow reruns
  - Placeholder rows may coexist with fully populated rows
- **Sub-workflow reference:** None.

#### 16) Wait4
- **Type and technical role:** `Wait`; adds a delay before reading back inserted rows.
- **Configuration choices:** Wait amount `3`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Inserted All Links in sheets`; output to `Get Rows from All links`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Delay may be insufficient for propagation in some environments.
- **Sub-workflow reference:** None.

#### 17) Get Rows from All links
- **Type and technical role:** `Google Sheets`; fetches the just-inserted row(s) using `Page_URL`.
- **Configuration choices:**  
  - Sheet: `YOUR_SEO_DATA`
  - Filter: `Page_URL` equals current `Page_URL`
- **Key expressions or variables used:** `={{ $json.Page_URL }}`
- **Input and output connections:** Input from `Wait4`; output to `Loop Over Items`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - If duplicates exist, multiple rows may match
  - If append hasn't propagated, fetch may return nothing
- **Sub-workflow reference:** None.

#### 18) All Links (Competitor Website)
- **Type and technical role:** `Code`; same link-discovery logic for competitor.
- **Configuration choices:** Same keyword list and URL normalization as target-site branch.
- **Key expressions or variables used:**  
  - `$('Edit Fields').first().json.competitor_site`
  - `$json.all_links || []`
- **Input and output connections:** Input from `HTML (for Competitor Website)`; output to `Inserted all Links (Competitor Website)`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Same heuristic limitations as target-site branch.
- **Sub-workflow reference:** None.

#### 19) Inserted all Links (Competitor Website)
- **Type and technical role:** `Google Sheets`; appends competitor discovered URLs as placeholder rows.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet: `COMPETITOR_SEO_DATA`
  - Only `Page_URL` populated
- **Key expressions or variables used:** `={{ $json.page_url }}`
- **Input and output connections:** Input from `All Links (Competitor Website)`; output to `Wait2`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:** Duplicate row risk identical to target branch.
- **Sub-workflow reference:** None.

#### 20) Wait2
- **Type and technical role:** `Wait`; pauses before reading competitor placeholder rows.
- **Configuration choices:** No explicit amount shown.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Inserted all Links (Competitor Website)`; output to `Get row(s) in sheet2`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - No explicit wait amount may cause immediate resume behavior depending on node defaults
- **Sub-workflow reference:** None.

#### 21) Get row(s) in sheet2
- **Type and technical role:** `Google Sheets`; fetches competitor sheet rows by `Page_URL`.
- **Configuration choices:**  
  - Sheet: `COMPETITOR_SEO_DATA`
  - Filter: `Page_URL` equals current `Page_URL`
- **Key expressions or variables used:** `={{ $json.Page_URL }}`
- **Input and output connections:** Input from `Wait2`; output to `Loop Over Items1`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Duplicate row returns
  - Eventual consistency delay
- **Sub-workflow reference:** None.

---

## 2.4 Page-by-Page Scraping for Your Website

**Overview:**  
This block iterates through target-site page URLs, downloads each page, extracts the same SEO fields used for the homepage, computes metrics, and updates the corresponding row in the sheet.

**Nodes Involved:**  
- Loop Over Items
- HTTP Request2
- HTML (All Links My Web)
- All Links Data (My Website)
- Wait
- Update row in My Sheet
- Wait5
- Get All data from My Sheet

### Node Details

#### 22) Loop Over Items
- **Type and technical role:** `Split In Batches`; controls iteration over target-site page rows.
- **Configuration choices:** Default options.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `Get Rows from All links`
  - Iteration output to `HTTP Request2`
  - Completion/next-cycle path via `All Links Data (My Website)` back to itself
  - Additional branch to `Wait`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Incorrect batch iteration behavior if upstream returns duplicates
  - Parallel side path may create timing complexity
- **Sub-workflow reference:** None.

#### 23) HTTP Request2
- **Type and technical role:** `HTTP Request`; fetches a discovered target-site page.
- **Configuration choices:**  
  - URL: `={{ $('Get Rows from All links').item.json.Page_URL }}`
  - `onError: continueRegularOutput`
- **Key expressions or variables used:** `={{ $('Get Rows from All links').item.json.Page_URL }}`
- **Input and output connections:** Input from `Loop Over Items`; output to `HTML (All Links My Web)`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Broken URLs, forbidden pages, timeout
  - Because errors continue, downstream nodes may receive incomplete payloads
- **Sub-workflow reference:** None.

#### 24) HTML (All Links My Web)
- **Type and technical role:** `HTML`; extracts SEO fields from each target-site internal page.
- **Configuration choices:** Same extraction map as homepage extractor.
- **Key expressions or variables used:** CSS selectors.
- **Input and output connections:** Input from `HTTP Request2`; output to `All Links Data (My Website)`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - `continueRegularOutput` on this node as well
  - Empty HTML still yields sparse data
- **Sub-workflow reference:** None.

#### 25) All Links Data (My Website)
- **Type and technical role:** `Code`; converts extracted internal-page HTML fields into normalized SEO records.
- **Configuration choices:**  
  - Uses `$input.first().json.page_url` as `Page_URL`
  - Assigns `Website` from original target URL
  - Computes word count, H2 cleanup, link counts, collected date
- **Key expressions or variables used:**  
  - `$('Edit Fields').first().json.your_site`
  - `$input.first().json.page_url`
- **Input and output connections:** Input from `HTML (All Links My Web)`; output back to `Loop Over Items`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Same `images`/`image` mismatch
  - Reliance on `$input.first().json.page_url` may fail if not preserved as expected
- **Sub-workflow reference:** None.

#### 26) Wait
- **Type and technical role:** `Wait`; delay branch before update.
- **Configuration choices:** Wait amount `2`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Loop Over Items`; output to `Update row in My Sheet`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Timing race with data generation path.
- **Sub-workflow reference:** None.

#### 27) Update row in My Sheet
- **Type and technical role:** `Google Sheets`; updates the target-site row matched by `Page_URL`.
- **Configuration choices:**  
  - Operation: `update`
  - Sheet: `YOUR_SEO_DATA`
  - Matching column: `Page_URL`
  - Writes SEO metrics into the placeholder row
- **Key expressions or variables used:**  
  - `={{ $('Get Rows from All links').item.json.Page_URL }}`
  - Other values from current JSON
- **Input and output connections:** Input from `Wait`; output to `Wait5`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - If multiple rows share `Page_URL`, update behavior may be ambiguous
  - If the update executes before extracted data is ready, fields may be empty depending on item pairing
- **Sub-workflow reference:** None.

#### 28) Wait5
- **Type and technical role:** `Wait`; delay before reading back fully updated target rows.
- **Configuration choices:** Wait amount `3`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Update row in My Sheet`; output to `Get All data from My Sheet`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Delay may still be insufficient if Sheets API writes lag.
- **Sub-workflow reference:** None.

#### 29) Get All data from My Sheet
- **Type and technical role:** `Google Sheets`; fetches the updated target record by `Page_URL`.
- **Configuration choices:**  
  - Sheet: `YOUR_SEO_DATA`
  - Filter on `Page_URL`
- **Key expressions or variables used:** `={{ $json.Page_URL }}`
- **Input and output connections:** Input from `Wait5`; output to `Merge`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:** Duplicate rows may return multiple items.
- **Sub-workflow reference:** None.

---

## 2.5 Page-by-Page Scraping for Competitor Website

**Overview:**  
This block mirrors the target-site internal-page scraping logic for the competitor website. It iterates discovered URLs, scrapes each page, updates the competitor sheet, and re-reads the updated records for merging.

**Nodes Involved:**  
- Loop Over Items1
- HTTP Request
- HTML (All Links Competitor Website)
- All Links Data (Competitor Website)
- Wait1
- Update row in Competitor Sheet
- Wait6
- Get All data from Competitor Sheet

### Node Details

#### 30) Loop Over Items1
- **Type and technical role:** `Split In Batches`; iterates through competitor page rows.
- **Configuration choices:** Default options.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `Get row(s) in sheet2`
  - Iteration output to `HTTP Request`
  - Secondary branch to `Wait1`
  - Loopback from `All Links Data (Competitor Website)`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:** Same iteration and pairing concerns as target-site loop.
- **Sub-workflow reference:** None.

#### 31) HTTP Request
- **Type and technical role:** `HTTP Request`; fetches each competitor internal page.
- **Configuration choices:**  
  - URL: `={{ $('Get row(s) in sheet2').item.json.Page_URL }}`
  - `onError: continueRegularOutput`
- **Key expressions or variables used:** `={{ $('Get row(s) in sheet2').item.json.Page_URL }}`
- **Input and output connections:** Input from `Loop Over Items1`; output to `HTML (All Links Competitor Website)`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:** Same as other HTTP requests.
- **Sub-workflow reference:** None.

#### 32) HTML (All Links Competitor Website)
- **Type and technical role:** `HTML`; extracts SEO fields from competitor internal pages.
- **Configuration choices:** Same extraction map as other HTML nodes.
- **Key expressions or variables used:** CSS selectors only.
- **Input and output connections:** Input from `HTTP Request`; output to `All Links Data (Competitor Website)`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Sparse output on script-heavy pages.
- **Sub-workflow reference:** None.

#### 33) All Links Data (Competitor Website)
- **Type and technical role:** `Code`; converts competitor internal-page HTML into structured SEO metrics.
- **Configuration choices:** Similar to target-site equivalent.
- **Key expressions or variables used:**  
  - `$('Edit Fields').first().json.your_site`
  - `$input.first().json.page_url`
- **Input and output connections:** Input from `HTML (All Links Competitor Website)`; output back to `Loop Over Items1`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Likely bug: output field `Website` is set to `your_site`, not competitor site
  - This may affect later dataset classification
  - Same `images`/`image` mismatch issue
- **Sub-workflow reference:** None.

#### 34) Wait1
- **Type and technical role:** `Wait`; delay branch before updating competitor rows.
- **Configuration choices:** Wait amount `2`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Loop Over Items1`; output to `Update row in Competitor Sheet`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Same timing concerns as target branch.
- **Sub-workflow reference:** None.

#### 35) Update row in Competitor Sheet
- **Type and technical role:** `Google Sheets`; updates the competitor placeholder row matched by `Page_URL`.
- **Configuration choices:**  
  - Operation: `update`
  - Matching column: `Page_URL`
  - Sheet: `COMPETITOR_SEO_DATA`
- **Key expressions or variables used:**  
  - `={{ $('Get row(s) in sheet2').item.json.Page_URL }}`
- **Input and output connections:** Input from `Wait1`; output to `Wait6`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Duplicate `Page_URL` ambiguity
  - Possible item pairing issue if update branch does not receive extracted fields
- **Sub-workflow reference:** None.

#### 36) Wait6
- **Type and technical role:** `Wait`; delay before readback.
- **Configuration choices:** Wait amount `3`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Update row in Competitor Sheet`; output to `Get All data from Competitor Sheet`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Same as other wait nodes.
- **Sub-workflow reference:** None.

#### 37) Get All data from Competitor Sheet
- **Type and technical role:** `Google Sheets`; reads updated competitor rows by `Page_URL`.
- **Configuration choices:**  
  - Sheet: `COMPETITOR_SEO_DATA`
  - Filter on `Page_URL`
- **Key expressions or variables used:** `={{ $json.Page_URL }}`
- **Input and output connections:** Input from `Wait6`; output to `Merge`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:** Duplicate row return risk.
- **Sub-workflow reference:** None.

---

## 2.6 Dataset Consolidation and AI Analysis

**Overview:**  
This block merges both website datasets, normalizes them into one payload, sends them to Gemini, and parses the AI response into strict JSON for reporting.

**Nodes Involved:**  
- Merge
- Code in JavaScript
- Message a model
- Code in JavaScript1

### Node Details

#### 38) Merge
- **Type and technical role:** `Merge`; combines target and competitor streams.
- **Configuration choices:** Default merge settings.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input 1 from `Get All data from My Sheet`
  - Input 2 from `Get All data from Competitor Sheet`
  - Output to `Code in JavaScript`
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:**  
  - Default merge mode matters; behavior should be verified in UI
  - Unequal item counts may produce partial or unexpected merged streams
- **Sub-workflow reference:** None.

#### 39) Code in JavaScript
- **Type and technical role:** `Code`; separates merged items into two arrays.
- **Configuration choices:**  
  - Pushes items with `Website` into `yourData`
  - Pushes items with `Competitor_Website` into `competitorData`
  - Outputs:
    - `your_website_data`
    - `competitor_website_data`
- **Key expressions or variables used:** `$input.all()`
- **Input and output connections:** Input from `Merge`; output to `Message a model`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Competitor internal-page rows may not contain `Competitor_Website`, only homepage rows do
  - Due to earlier inconsistency, competitor data may be under-collected
- **Sub-workflow reference:** None.

#### 40) Message a model
- **Type and technical role:** `Google Gemini` via LangChain; performs AI-driven SEO comparison.
- **Configuration choices:**  
  - Model: `models/gemini-2.5-flash`
  - Prompt instructs the model to act as an SEO expert
  - Includes:
    - website URLs
    - required analysis categories
    - strict JSON response schema
    - embedded datasets from both arrays
- **Key expressions or variables used:**  
  - `{{ $('Edit Fields').item.json.your_site }}`
  - `{{ $('Edit Fields').item.json.competitor_site }}`
  - `{{ $json.your_website_data }}`
  - `{{ $json.competitor_website_data }}`
  - `{{ $now }}`
- **Input and output connections:** Input from `Code in JavaScript`; output to `Code in JavaScript1`.
- **Version-specific requirements:** Type version `1`; requires Google Gemini credentials/API access.
- **Edge cases or potential failure types:**  
  - Credential or quota errors
  - Token/context size limits if datasets become large
  - Model may still emit markdown or malformed JSON
  - Hallucinated schema deviations despite prompt
- **Sub-workflow reference:** None.

#### 41) Code in JavaScript1
- **Type and technical role:** `Code`; cleans and parses Gemini output.
- **Configuration choices:**  
  - Reads `content.parts[0].text`
  - Removes ```json fences and markdown bold markers
  - Extracts first JSON object with regex
  - Parses it
  - Throws explicit errors if parsing fails
- **Key expressions or variables used:** `$json.content.parts[0].text`
- **Input and output connections:** Input from `Message a model`; output to `Append row in sheet`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If Gemini output shape changes, `content.parts[0].text` may be undefined
  - Regex may capture wrong object if extra braces exist in text
  - Throws hard error if no valid JSON found
- **Sub-workflow reference:** None.

---

## 2.7 Reporting and Status Update

**Overview:**  
This block stores the AI-produced SEO gap report and marks the original input row as completed.

**Nodes Involved:**  
- Append row in sheet
- Update row in sheet

### Node Details

#### 42) Append row in sheet
- **Type and technical role:** `Google Sheets`; writes the final structured SEO gap report.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet: `SEO_GAP_REPORT`
  - Stores fields such as `Keyword_Gap`, `Content_Gap`, `Technical_SEO_Gap`, `Suggested_Keywords`, and `SEO_Improvement_Plan`
  - `Analysis_Date` uses `{{$now}}`
- **Key expressions or variables used:**  
  - `={{ $json.ID }}`
  - `={{ $now }}`
  - Other parsed JSON fields
- **Input and output connections:** Input from `Code in JavaScript1`; output to `Update row in sheet`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - If Gemini omitted `ID`, input row linkage may be lost
  - Append duplicates on reruns
- **Sub-workflow reference:** None.

#### 43) Update row in sheet
- **Type and technical role:** `Google Sheets`; marks the input record as processed.
- **Configuration choices:**  
  - Operation: `update`
  - Sheet: `INPUT_WEBSITES`
  - Matching column: `ID`
  - Sets `Status` to `DONE`
  - Sets `Processed_Date` to `{{$now}}`
- **Key expressions or variables used:**  
  - `={{ $json.ID }}`
  - `={{ $now }}`
- **Input and output connections:** Input from `Append row in sheet`; terminal node.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - If parsed AI output lacks the original `ID`, update may fail or affect wrong row if blank IDs exist
  - Better practice would be to source `ID` from the original input node instead of the model output
- **Sub-workflow reference:** None.

---

## 2.8 Documentation / Visual Annotation Nodes

**Overview:**  
These nodes are non-executable sticky notes used for human-readable documentation inside the canvas. They provide context, setup instructions, and block labels.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### 44) Sticky Note
- **Type and technical role:** `Sticky Note`; visual documentation.
- **Configuration choices:** General workflow explanation, setup steps, customization tips.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 45) Sticky Note1
- **Type and technical role:** `Sticky Note`; labels input block.
- **Configuration choices:** Notes that input is read from Sheets and filtered to `NEW`.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 46) Sticky Note2
- **Type and technical role:** `Sticky Note`; labels scraping/data collection area.
- **Configuration choices:** Notes that SEO data, pages, and structure are scraped.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 47) Sticky Note3
- **Type and technical role:** `Sticky Note`; labels AI analysis block.
- **Configuration choices:** Notes Gemini-based comparison and report generation.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual workflow start |  | Get row(s) in sheet | # **Input & Validation** #<br><br>Reads URLs from Google Sheets<br>Filters only NEW rows for processing |
| Get row(s) in sheet | Google Sheets | Read input website pairs from input sheet | When clicking ‘Execute workflow’ | If | # **Input & Validation** #<br><br>Reads URLs from Google Sheets<br>Filters only NEW rows for processing |
| If | If | Filter rows where Status = NEW | Get row(s) in sheet | Edit Fields | # **Input & Validation** #<br><br>Reads URLs from Google Sheets<br>Filters only NEW rows for processing |
| Edit Fields | Set | Normalize field names for downstream use | If | My Website HTTP, HTTP (for Competitor) | # **Input & Validation** #<br><br>Reads URLs from Google Sheets<br>Filters only NEW rows for processing |
| My Website HTTP | HTTP Request | Fetch target homepage HTML | Edit Fields | HTML (for My website) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| HTML (for My website) | HTML | Extract SEO elements from target homepage | My Website HTTP | My Website Data, All Links (My Website) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| My Website Data | Code | Build structured homepage SEO record for target site | HTML (for My website) | Append First Row of My website | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Append First Row of My website | Google Sheets | Append target homepage SEO row | My Website Data |  | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| All Links (My Website) | Code | Discover important internal target URLs | HTML (for My website) | Wait3 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Wait3 | Wait | Delay before appending target URLs | All Links (My Website) | Inserted All Links in sheets | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Inserted All Links in sheets | Google Sheets | Append target internal page URLs as placeholders | Wait3 | Wait4 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Wait4 | Wait | Delay before reading target placeholder rows | Inserted All Links in sheets | Get Rows from All links | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Get Rows from All links | Google Sheets | Read target rows by Page_URL | Wait4 | Loop Over Items | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Loop Over Items | Split In Batches | Iterate target page rows | Get Rows from All links, All Links Data (My Website) | Wait, HTTP Request2 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| HTTP Request2 | HTTP Request | Fetch target internal page HTML | Loop Over Items | HTML (All Links My Web) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| HTML (All Links My Web) | HTML | Extract SEO elements from target internal pages | HTTP Request2 | All Links Data (My Website) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| All Links Data (My Website) | Code | Build structured target internal-page SEO rows | HTML (All Links My Web) | Loop Over Items | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Wait | Wait | Delay before updating target sheet row | Loop Over Items | Update row in My Sheet | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Update row in My Sheet | Google Sheets | Update target page row with extracted metrics | Wait | Wait5 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Wait5 | Wait | Delay before re-reading updated target data | Update row in My Sheet | Get All data from My Sheet | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Get All data from My Sheet | Google Sheets | Read final target page data for merge | Wait5 | Merge | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| HTTP (for Competitor) | HTTP Request | Fetch competitor homepage HTML | Edit Fields | HTML (for Competitor Website) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| HTML (for Competitor Website) | HTML | Extract SEO elements from competitor homepage | HTTP (for Competitor) | Competitor Data, All Links (Competitor Website) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Competitor Data | Code | Build structured homepage SEO record for competitor site | HTML (for Competitor Website) | Append in Competitor Sheet | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Append in Competitor Sheet | Google Sheets | Append competitor homepage SEO row | Competitor Data |  | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| All Links (Competitor Website) | Code | Discover important internal competitor URLs | HTML (for Competitor Website) | Inserted all Links (Competitor Website) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Inserted all Links (Competitor Website) | Google Sheets | Append competitor internal page URLs as placeholders | All Links (Competitor Website) | Wait2 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Wait2 | Wait | Delay before reading competitor placeholder rows | Inserted all Links (Competitor Website) | Get row(s) in sheet2 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Get row(s) in sheet2 | Google Sheets | Read competitor rows by Page_URL | Wait2 | Loop Over Items1 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Loop Over Items1 | Split In Batches | Iterate competitor page rows | Get row(s) in sheet2, All Links Data (Competitor Website) | Wait1, HTTP Request | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| HTTP Request | HTTP Request | Fetch competitor internal page HTML | Loop Over Items1 | HTML (All Links Competitor Website) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| HTML (All Links Competitor Website) | HTML | Extract SEO elements from competitor internal pages | HTTP Request | All Links Data (Competitor Website) | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| All Links Data (Competitor Website) | Code | Build structured competitor internal-page SEO rows | HTML (All Links Competitor Website) | Loop Over Items1 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Wait1 | Wait | Delay before updating competitor sheet row | Loop Over Items1 | Update row in Competitor Sheet | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Update row in Competitor Sheet | Google Sheets | Update competitor page row with extracted metrics | Wait1 | Wait6 | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Wait6 | Wait | Delay before re-reading updated competitor data | Update row in Competitor Sheet | Get All data from Competitor Sheet | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Get All data from Competitor Sheet | Google Sheets | Read final competitor page data for merge | Wait6 | Merge | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Merge | Merge | Combine target and competitor streams | Get All data from My Sheet, Get All data from Competitor Sheet | Code in JavaScript | # **AI SEO Analysis** #<br><br>Compares both datasets using Gemini<br>Generates SEO gap report |
| Code in JavaScript | Code | Separate merged items into target and competitor arrays | Merge | Message a model | # **AI SEO Analysis** #<br><br>Compares both datasets using Gemini<br>Generates SEO gap report |
| Message a model | Google Gemini | Generate AI SEO gap analysis | Code in JavaScript | Code in JavaScript1 | # **AI SEO Analysis** #<br><br>Compares both datasets using Gemini<br>Generates SEO gap report |
| Code in JavaScript1 | Code | Parse Gemini text output into JSON | Message a model | Append row in sheet | # **AI SEO Analysis** #<br><br>Compares both datasets using Gemini<br>Generates SEO gap report |
| Append row in sheet | Google Sheets | Append final SEO gap report | Code in JavaScript1 | Update row in sheet | # **AI SEO Analysis** #<br><br>Compares both datasets using Gemini<br>Generates SEO gap report |
| Update row in sheet | Google Sheets | Mark original input row as DONE | Append row in sheet |  | # **AI SEO Analysis** #<br><br>Compares both datasets using Gemini<br>Generates SEO gap report |
| Sticky Note | Sticky Note | Canvas documentation |  |  | # SEO Content Gap Analyzer<br><br>### How it works<br><br>This workflow compares your website with competitor websites to identify SEO gaps. It starts by reading URLs from Google Sheets and filtering only new entries. It then scrapes key SEO elements such as titles, meta descriptions, headings, and page content from both sites. The workflow expands into important internal pages (like services, blogs, and contact pages) to ensure deeper analysis.<br><br>All collected data is structured and stored before being sent to Google Gemini AI. The AI analyzes differences between both websites to identify keyword gaps, missing content, technical SEO issues, and opportunities for improvement. Finally, it generates a structured SEO report including suggested keywords, blog topics, and an improvement plan, which is saved back into Google Sheets.<br><br>### Setup steps<br><br>1. Connect your Google Sheets account.<br>2. Add your website and competitor URLs in the input sheet.<br>3. Connect your Google Gemini API credentials.<br>4. Ensure sheets for output data and reports are properly configured.<br><br>### Customization tips<br><br>- Add more competitors for broader analysis.<br>- Adjust page filtering logic for deeper crawling.<br>- Modify the AI prompt for different SEO outputs. |
| Sticky Note1 | Sticky Note | Input block annotation |  |  | # **Input & Validation** #<br><br>Reads URLs from Google Sheets<br>Filters only NEW rows for processing |
| Sticky Note2 | Sticky Note | Data collection block annotation |  |  | # **Data Collection** #<br><br>Scrapes SE  O data from both websites<br>Extracts pages, content, and structure |
| Sticky Note3 | Sticky Note | AI analysis block annotation |  |  | # **AI SEO Analysis** #<br><br>Compares both datasets using Gemini<br>Generates SEO gap report |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence in n8n.

## Prerequisites
1. Create Google Sheets credentials in n8n.
2. Create Google Gemini credentials in n8n for the `Google Gemini` LangChain node.
3. Create a Google Spreadsheet named, for example, `SEO_Workflow`.
4. Add these sheets and columns:

### 4.1 INPUT_WEBSITES
Columns:
- `ID`
- `Your_Website`
- `Competitor_Website`
- `Status`
- `Created_Date`
- `Processed_Date`

Recommended values:
- `Status` should be `NEW` for rows awaiting processing.

### 4.2 YOUR_SEO_DATA
Columns:
- `ID`
- `Website`
- `Page_URL`
- `Page_Title`
- `Meta_Description`
- `H1`
- `H2`
- `Word_Count`
- `Internal_Links`
- `External_Links`
- `Images`
- `Collected_Date`
- `All Links` optional if desired

### 4.3 COMPETITOR_SEO_DATA
Columns:
- `ID`
- `Competitor_Website`
- `Page_URL`
- `Page_Title`
- `Meta_Description`
- `H1`
- `H2`
- `Word_Count`
- `Internal_Links`
- `External_Links`
- `Images`
- `Collected_Date`

### 4.4 SEO_GAP_REPORT
Columns:
- `ID`
- `Your_Website`
- `Competitor_Website`
- `Keyword_Gap`
- `Content_Gap`
- `Technical_SEO_Gap`
- `Missing_Blog_Topics`
- `Suggested_Keywords`
- `Suggested_Blog_Topics`
- `SEO_Improvement_Plan`
- `Analysis_Date`

---

## Step-by-step build

1. **Create a `Manual Trigger` node** named `When clicking ‘Execute workflow’`.

2. **Add a `Google Sheets` node** named `Get row(s) in sheet`.
   - Credential: your Google Sheets credential
   - Spreadsheet: `SEO_Workflow`
   - Sheet: `INPUT_WEBSITES`
   - Operation: read/get rows

3. **Connect** `When clicking ‘Execute workflow’ -> Get row(s) in sheet`.

4. **Add an `If` node** named `If`.
   - Condition: string equals
   - Left value: `{{ $json.Status }}`
   - Right value: `NEW`

5. **Connect** `Get row(s) in sheet -> If`.

6. **Add a `Set` node** named `Edit Fields`.
   - Add fields:
     - `your_site` = `{{ $json.Your_Website }}`
     - `competitor_site` = `{{ $json.Competitor_Website }}`
     - `row_number` = `{{ $json.row_number }}`
   - Keep only these if desired, though current workflow uses external references to the original row elsewhere.

7. **Connect** `If (true output) -> Edit Fields`.

---

## Build the target homepage branch

8. **Add an `HTTP Request` node** named `My Website HTTP`.
   - URL: `{{ $json.your_site }}`
   - Response: default/text-compatible HTML response

9. **Connect** `Edit Fields -> My Website HTTP`.

10. **Add an `HTML` node** named `HTML (for My website)`.
   - Operation: Extract HTML Content
   - Add extraction values:
     - `title` with selector `title`
     - `meta_description` with selector `meta[name="description"]`, return attribute `content`
     - `h1` with selector `h1`
     - `h2` with selector `h2`, return array
     - `content` with selector `p`, return array
     - `internal_links` with selector `a[href^="/"]`, return attribute `href`, return array
     - `external_links` with selector `a[href^="http"]`, return attribute `href`, return array
     - `image` with selector `img`, return attribute `src`, return array
     - `all_links` with selector `a`, return attribute `href`, return array

11. **Connect** `My Website HTTP -> HTML (for My website)`.

12. **Add a `Code` node** named `My Website Data`.
   - Paste the homepage transformation logic from the workflow.
   - Recommended correction: use `item.image` instead of `item.images` when counting images.

13. **Connect** `HTML (for My website) -> My Website Data`.

14. **Add a `Google Sheets` node** named `Append First Row of My website`.
   - Operation: Append
   - Sheet: `YOUR_SEO_DATA`
   - Map:
     - `ID` = `{{ $('Get row(s) in sheet').item.json.ID }}`
     - `Website` = `{{ $json.Website }}`
     - `Page_URL` = `{{ $json.Page_URL }}`
     - `Page_Title` = `{{ $json.Page_Title }}`
     - `Meta_Description` = `{{ $json.Meta_Description }}`
     - `H1` = `{{ $json.H1 }}`
     - `H2` = `{{ $json.H2 }}`
     - `Word_Count` = `{{ $json.Word_Count }}`
     - `Internal_Links` = `{{ $json.Internal_Links }}`
     - `External_Links` = `{{ $json.External_Links }}`
     - `Images` = `{{ $json.Images }}`
     - `Collected_Date` = `{{ $json.Collected_Date }}`

15. **Connect** `My Website Data -> Append First Row of My website`.

---

## Build target internal-link discovery

16. **Add a `Code` node** named `All Links (My Website)`.
   - Paste the link-filtering code from the workflow.
   - It should:
     - normalize base URL,
     - extract important pages,
     - keep homepage,
     - return items with `page_url`.

17. **Connect** `HTML (for My website) -> All Links (My Website)`.

18. **Add a `Wait` node** named `Wait3`.
   - Amount: `3`  
   - Use the same default unit as in your n8n instance.

19. **Connect** `All Links (My Website) -> Wait3`.

20. **Add a `Google Sheets` node** named `Inserted All Links in sheets`.
   - Operation: Append
   - Sheet: `YOUR_SEO_DATA`
   - Map `Page_URL` = `{{ $json.page_url }}`

21. **Connect** `Wait3 -> Inserted All Links in sheets`.

22. **Add a `Wait` node** named `Wait4`.
   - Amount: `3`

23. **Connect** `Inserted All Links in sheets -> Wait4`.

24. **Add a `Google Sheets` node** named `Get Rows from All links`.
   - Operation: Get rows / read rows with filter
   - Sheet: `YOUR_SEO_DATA`
   - Filter:
     - Column: `Page_URL`
     - Value: `{{ $json.Page_URL }}`

   Note: If your append node emits `page_url`, you may need to adapt this expression to `{{ $json.page_url }}` depending on actual output behavior.

25. **Connect** `Wait4 -> Get Rows from All links`.

---

## Build target page loop

26. **Add a `Split In Batches` node** named `Loop Over Items`.

27. **Connect** `Get Rows from All links -> Loop Over Items`.

28. **Add an `HTTP Request` node** named `HTTP Request2`.
   - URL: `{{ $('Get Rows from All links').item.json.Page_URL }}`
   - On Error: continue regular output

29. **Connect** `Loop Over Items -> HTTP Request2`.

30. **Add an `HTML` node** named `HTML (All Links My Web)`.
   - Same extraction configuration as the homepage HTML node
   - On Error: continue regular output

31. **Connect** `HTTP Request2 -> HTML (All Links My Web)`.

32. **Add a `Code` node** named `All Links Data (My Website)`.
   - Paste the code from the workflow.
   - Recommended correction: preserve page URL explicitly and use `image` key consistently.

33. **Connect** `HTML (All Links My Web) -> All Links Data (My Website)`.

34. **Connect loopback** `All Links Data (My Website) -> Loop Over Items` so the split node continues iteration.

35. **Add a `Wait` node** named `Wait`.
   - Amount: `2`

36. **Connect** `Loop Over Items -> Wait`.

37. **Add a `Google Sheets` node** named `Update row in My Sheet`.
   - Operation: Update
   - Sheet: `YOUR_SEO_DATA`
   - Matching column: `Page_URL`
   - Map all page metrics
   - `Page_URL` = `{{ $('Get Rows from All links').item.json.Page_URL }}`

38. **Connect** `Wait -> Update row in My Sheet`.

39. **Add a `Wait` node** named `Wait5`.
   - Amount: `3`

40. **Connect** `Update row in My Sheet -> Wait5`.

41. **Add a `Google Sheets` node** named `Get All data from My Sheet`.
   - Read rows from `YOUR_SEO_DATA`
   - Filter `Page_URL = {{ $json.Page_URL }}`

42. **Connect** `Wait5 -> Get All data from My Sheet`.

---

## Build competitor homepage branch

43. **Add an `HTTP Request` node** named `HTTP (for Competitor)`.
   - URL: `{{ $('Edit Fields').item.json.competitor_site }}`
   - Response format: text

44. **Connect** `Edit Fields -> HTTP (for Competitor)`.

45. **Add an `HTML` node** named `HTML (for Competitor Website)`.
   - Same extraction settings as the target homepage HTML node

46. **Connect** `HTTP (for Competitor) -> HTML (for Competitor Website)`.

47. **Add a `Code` node** named `Competitor Data`.
   - Paste the code from the workflow.
   - Recommended correction: use `item.image` for image count.

48. **Connect** `HTML (for Competitor Website) -> Competitor Data`.

49. **Add a `Google Sheets` node** named `Append in Competitor Sheet`.
   - Operation: Append
   - Sheet: `COMPETITOR_SEO_DATA`
   - Map:
     - `ID` = `{{ $('Get row(s) in sheet').item.json.ID }}`
     - `Competitor_Website` = `{{ $json.Website }}`
     - and the other SEO fields

50. **Connect** `Competitor Data -> Append in Competitor Sheet`.

---

## Build competitor internal-link discovery

51. **Add a `Code` node** named `All Links (Competitor Website)`.
   - Paste the competitor link-filtering code from the workflow.

52. **Connect** `HTML (for Competitor Website) -> All Links (Competitor Website)`.

53. **Add a `Google Sheets` node** named `Inserted all Links (Competitor Website)`.
   - Operation: Append
   - Sheet: `COMPETITOR_SEO_DATA`
   - Map `Page_URL = {{ $json.page_url }}`

54. **Connect** `All Links (Competitor Website) -> Inserted all Links (Competitor Website)`.

55. **Add a `Wait` node** named `Wait2`.
   - In the source workflow it has no visible amount; set a small delay such as `2` or `3` to make behavior deterministic.

56. **Connect** `Inserted all Links (Competitor Website) -> Wait2`.

57. **Add a `Google Sheets` node** named `Get row(s) in sheet2`.
   - Read rows from `COMPETITOR_SEO_DATA`
   - Filter `Page_URL = {{ $json.Page_URL }}`  
   Again, depending on actual append output, `{{ $json.page_url }}` may be safer.

58. **Connect** `Wait2 -> Get row(s) in sheet2`.

---

## Build competitor page loop

59. **Add a `Split In Batches` node** named `Loop Over Items1`.

60. **Connect** `Get row(s) in sheet2 -> Loop Over Items1`.

61. **Add an `HTTP Request` node** named `HTTP Request`.
   - URL: `{{ $('Get row(s) in sheet2').item.json.Page_URL }}`
   - On Error: continue regular output

62. **Connect** `Loop Over Items1 -> HTTP Request`.

63. **Add an `HTML` node** named `HTML (All Links Competitor Website)`.
   - Same extraction config as other HTML nodes
   - On Error: continue regular output

64. **Connect** `HTTP Request -> HTML (All Links Competitor Website)`.

65. **Add a `Code` node** named `All Links Data (Competitor Website)`.
   - Paste the code from the workflow.
   - Recommended correction: set `Website` or `Competitor_Website` to the competitor URL, not `your_site`.

66. **Connect** `HTML (All Links Competitor Website) -> All Links Data (Competitor Website)`.

67. **Connect loopback** `All Links Data (Competitor Website) -> Loop Over Items1`.

68. **Add a `Wait` node** named `Wait1`.
   - Amount: `2`

69. **Connect** `Loop Over Items1 -> Wait1`.

70. **Add a `Google Sheets` node** named `Update row in Competitor Sheet`.
   - Operation: Update
   - Sheet: `COMPETITOR_SEO_DATA`
   - Matching column: `Page_URL`
   - Map extracted fields
   - `Page_URL = {{ $('Get row(s) in sheet2').item.json.Page_URL }}`

71. **Connect** `Wait1 -> Update row in Competitor Sheet`.

72. **Add a `Wait` node** named `Wait6`.
   - Amount: `3`

73. **Connect** `Update row in Competitor Sheet -> Wait6`.

74. **Add a `Google Sheets` node** named `Get All data from Competitor Sheet`.
   - Read rows from `COMPETITOR_SEO_DATA`
   - Filter `Page_URL = {{ $json.Page_URL }}`

75. **Connect** `Wait6 -> Get All data from Competitor Sheet`.

---

## Build merge and AI analysis

76. **Add a `Merge` node** named `Merge`.
   - Keep default mode unless you intentionally change merge behavior after testing.

77. **Connect**:
   - `Get All data from My Sheet -> Merge` input 1
   - `Get All data from Competitor Sheet -> Merge` input 2

78. **Add a `Code` node** named `Code in JavaScript`.
   - Paste the array-grouping code from the workflow.

79. **Connect** `Merge -> Code in JavaScript`.

80. **Add a `Google Gemini` node** named `Message a model`.
   - Credential: your Gemini credential
   - Model: `models/gemini-2.5-flash`
   - Message content: paste the prompt from the workflow, including:
     - your website URL
     - competitor URL
     - strict JSON output schema
     - both datasets

81. **Connect** `Code in JavaScript -> Message a model`.

82. **Add a `Code` node** named `Code in JavaScript1`.
   - Paste the parsing code from the workflow.
   - This strips markdown fences and parses the returned JSON object.

83. **Connect** `Message a model -> Code in JavaScript1`.

---

## Build report storage and final status update

84. **Add a `Google Sheets` node** named `Append row in sheet`.
   - Operation: Append
   - Sheet: `SEO_GAP_REPORT`
   - Map:
     - `ID`
     - `Your_Website`
     - `Competitor_Website`
     - `Keyword_Gap`
     - `Content_Gap`
     - `Technical_SEO_Gap`
     - `Missing_Blog_Topics`
     - `Suggested_Keywords`
     - `Suggested_Blog_Topics`
     - `SEO_Improvement_Plan`
     - `Analysis_Date = {{ $now }}`

85. **Connect** `Code in JavaScript1 -> Append row in sheet`.

86. **Add a `Google Sheets` node** named `Update row in sheet`.
   - Operation: Update
   - Sheet: `INPUT_WEBSITES`
   - Matching column: `ID`
   - Set:
     - `ID = {{ $json.ID }}`
     - `Status = DONE`
     - `Processed_Date = {{ $now }}`

87. **Connect** `Append row in sheet -> Update row in sheet`.

---

## Optional canvas annotations

88. Add sticky notes for:
   - general workflow description,
   - input/validation,
   - data collection,
   - AI SEO analysis.

---

## Recommended corrections during rebuild

89. **Fix image-count field mismatch** in all code nodes:
   - HTML nodes output `image`
   - code nodes reference `images`
   - either rename extractor key to `images` or update code to use `item.image`.

90. **Fix competitor internal-page data labeling** in `All Links Data (Competitor Website)`:
   - currently it sets `Website` using `your_site`
   - change it to the competitor site and/or populate `Competitor_Website`.

91. **Preserve original `ID` outside AI output**:
   - safer approach: attach `ID` from `Get row(s) in sheet` before Gemini and use that in reporting/status update
   - do not rely on the model to correctly echo the `ID`.

92. **Normalize URLs before matching by `Page_URL`**:
   - remove trailing slashes consistently
   - avoid duplicate rows from `https://site.com/page` vs `https://site.com/page/`.

93. **Consider deduplication before appending placeholder URLs**:
   - otherwise reruns may create repeated rows that complicate updates.

94. **Verify Wait node units in your n8n version**:
   - the JSON only shows numeric amounts, not explicit units in this export.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SEO Content Gap Analyzer | General workflow title/context |
| This workflow compares your website with competitor websites to identify SEO gaps. It starts by reading URLs from Google Sheets and filtering only new entries. It then scrapes key SEO elements such as titles, meta descriptions, headings, and page content from both sites. The workflow expands into important internal pages (like services, blogs, and contact pages) to ensure deeper analysis. | Workflow purpose |
| All collected data is structured and stored before being sent to Google Gemini AI. The AI analyzes differences between both websites to identify keyword gaps, missing content, technical SEO issues, and opportunities for improvement. Finally, it generates a structured SEO report including suggested keywords, blog topics, and an improvement plan, which is saved back into Google Sheets. | Workflow logic summary |
| Connect your Google Sheets account. | Setup note |
| Add your website and competitor URLs in the input sheet. | Setup note |
| Connect your Google Gemini API credentials. | Setup note |
| Ensure sheets for output data and reports are properly configured. | Setup note |
| Add more competitors for broader analysis. | Customization tip |
| Adjust page filtering logic for deeper crawling. | Customization tip |
| Modify the AI prompt for different SEO outputs. | Customization tip |

## Additional implementation observations
- There are **no sub-workflows** in this workflow.
- There is **one entry point**: the manual trigger.
- The workflow is highly dependent on **Google Sheets schema consistency**.
- The workflow uses **sheet append + later update by `Page_URL`**, which works but is vulnerable to duplicate URLs.
- The AI step is only as good as the completeness of the scraped page data; JS-heavy websites may produce weak extraction results.
- The current implementation contains a few likely logic bugs, especially around:
  - `image` vs `images`
  - competitor page data labeling
  - reliance on AI to return the correct `ID`
  - possible timing/pairing issues introduced by parallel wait/update paths.