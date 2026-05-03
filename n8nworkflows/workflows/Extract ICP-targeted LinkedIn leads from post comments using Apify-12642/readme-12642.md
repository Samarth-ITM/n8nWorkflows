Extract ICP-targeted LinkedIn leads from post comments using Apify

https://n8nworkflows.xyz/workflows/extract-icp-targeted-linkedin-leads-from-post-comments-using-apify-12642


# Extract ICP-targeted LinkedIn leads from post comments using Apify

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow collects a LinkedIn post URL plus ICP criteria (job titles and countries), scrapes up to ~1,000 post comments via Apify (with paging), deduplicates commenters by profile URL, enriches each unique commenter’s LinkedIn profile via Apify, filters enriched profiles against the ICP rules, and exports matching leads to a downloadable CSV.

**Target use cases:**
- Extract decision-makers who commented on a specific LinkedIn post.
- Build a quick lead list matching an ICP (role + geography).
- Produce a CSV for downstream import into a CRM or outreach tool.

### Logical blocks
1.1 **Input Reception (Form)**  
User submits LinkedIn post URL + selects job titles and countries.

1.2 **Scrape Comments with Pagination (Apify + loop)**  
Generates page requests and scrapes comments page-by-page.

1.3 **Deduplicate & Enrich Profiles (Dedup + loop + Apify enrichment)**  
Deduplicates by commenter profile URL, attaches ICP selections to each item, then enriches profiles one-by-one while preserving the ICP criteria through the loop.

1.4 **Filter & Export Leads (ICP match + aggregation + CSV)**  
Filters enriched profiles to those matching both title and country; aggregates results; formats and exports CSV.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Form)

**Overview:**  
Collects user inputs (LinkedIn post URL, target job titles, target countries) and starts the workflow execution in “responseMode: lastNode” so the form response is the output of the final node.

**Nodes involved:**
- **Sticky Note**
- **Sticky Note - Section 1**
- **ICP Lead Extractor Form**

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Provides workflow description, setup steps, requirements, customization ideas.
- **Connections:** None (informational only).
- **Edge cases:** None.

#### Node: Sticky Note - Section 1
- **Type / role:** Sticky Note (documents Block 1)
- **Configuration choices:** Explains the form’s purpose.
- **Connections:** None.
- **Edge cases:** None.

#### Node: ICP Lead Extractor Form
- **Type / role:** `formTrigger` (user-facing input entry point)
- **Key configuration:**
  - **Form title:** “LinkedIn ICP Lead Extractor”
  - **Button label:** “Extract Leads”
  - **Response mode:** `lastNode` (the form will return the CSV file output if successful)
  - **Fields:**
    - `LinkedIn Post URL` (required text)
    - `Target Job Titles` (required checkbox, predefined options)
    - `Target Countries` (required checkbox, predefined options)
- **Key variables/fields produced:** The submitted values are available in the trigger output (sometimes nested under `.data` depending on n8n version).
- **Output connections:** → **Prepare Pagination Requests**
- **Version notes:** Node typeVersion `2.4`. Output shape can vary (the workflow accounts for this later).
- **Potential failures / edge cases:**
  - Invalid/unsupported LinkedIn post URL format (will cause downstream Apify actor to return empty or error).
  - Empty selections are prevented by required fields.

---

### 2.2 Scrape Comments with Pagination

**Overview:**  
Creates up to 10 page requests and iterates them in batches. Each page triggers an Apify actor run that returns comment items. A loop node cycles through pages.

**Nodes involved:**
- **Sticky Note - Section 2**
- **Prepare Pagination Requests**
- **Loop Through Pages**
- **Scrape LinkedIn Comments**
- **Check for More Pages**

#### Node: Sticky Note - Section 2
- **Type / role:** Sticky Note (documents Block 2)
- **Connections:** None.

#### Node: Prepare Pagination Requests
- **Type / role:** `code` (builds an array of page request items)
- **Configuration choices (interpreted):**
  - Reads `postUrl` from incoming JSON field **`LinkedIn Post URL`**.
  - Generates `maxPages = 10` items, each containing:
    - `postUrl`
    - `pageNumber` (1..10)
    - `_originalFormData` copy of the entire input payload
- **Key expressions/variables:**
  - `$json['LinkedIn Post URL']`
  - `maxPages` controls upper bound (10 pages × 100 comments/page = up to ~1,000)
- **Input connections:** From **ICP Lead Extractor Form**
- **Output connections:** → **Loop Through Pages**
- **Potential failures / edge cases:**
  - If `LinkedIn Post URL` is missing/renamed, postUrl becomes `undefined` and Apify call likely fails.
  - Pagination is “pre-generated”; it does not truly stop early based on results (see notes under “Check for More Pages”).

#### Node: Loop Through Pages
- **Type / role:** `splitInBatches` (iteration controller over page requests)
- **Configuration choices:**
  - Uses default options (batch size default in n8n UI unless changed; not explicitly set here).
  - Has **two outgoing connections**:
    - **Index 1 output** → **Scrape LinkedIn Comments** (loop branch)
    - **Index 0 output** → **Remove Duplicate Commenters** (done branch)
- **Input connections:** From **Prepare Pagination Requests**
- **Output connections:**
  - Output 1 → **Scrape LinkedIn Comments**
  - Output 0 → **Remove Duplicate Commenters**
- **Version notes:** typeVersion `3`.
- **Edge cases / failure types:**
  - If batch size is >1, `Scrape LinkedIn Comments` receives multiple page items at once; ensure the Apify actor endpoint can handle that (this workflow assumes per-item execution).
  - The “done” output will fire after all items are processed.

#### Node: Scrape LinkedIn Comments
- **Type / role:** `httpRequest` (calls Apify actor to scrape comments)
- **Configuration choices:**
  - **POST** to Apify actor run endpoint:  
    `apimaestro~linkedin-post-comments-replies-engagements-scraper-no-cookies/run-sync-get-dataset-items`
  - **Timeout:** 600,000 ms (10 minutes)
  - **JSON body (built via expression):**
    - `postIds`: array containing `{{$json.postUrl}}`
    - `page_number`: `{{$json.pageNumber}}`
    - `sortOrder`: `"most recent"`
    - `limit`: `100`
  - **Authentication:** predefined credential type `apifyApi`
- **Input:** Each page request item (postUrl + pageNumber)
- **Output:** Comment items from Apify dataset items
- **Output connections:** → **Check for More Pages**
- **Potential failures / edge cases:**
  - Apify auth/credit limits, quota exhaustion, or 401/403.
  - Actor changes or deprecation.
  - LinkedIn anti-scraping changes may reduce returned items.
  - Large posts might require higher `maxPages` (cost tradeoff).

#### Node: Check for More Pages
- **Type / role:** `code` (intended to decide if pagination should continue)
- **Configuration choices (actual behavior):**
  - Computes:
    - `hasResults = items.length > 0 && items[0].json`
  - If no results or `< 100` items: returns items (comment says “stop pagination”).
  - Else returns items (comment says “continue loop”).
- **Connections:** → **Loop Through Pages**
- **Important note (logic caveat):**
  - This node does **not** control the SplitInBatches loop termination by itself; it always returns items and always routes back to **Loop Through Pages**. In n8n, the loop ends when SplitInBatches has no more input items, not because this node “stops” it.  
  - Practically: you still run all pre-generated pages (1..10). Pages returning 0 items simply contribute nothing.
- **Potential failures / edge cases:**
  - If Apify returns a non-array or unexpected shape, `items[0].json` may be undefined.
  - Could be improved by dynamically generating next pages only when needed.

---

### 2.3 Deduplicate & Enrich Profiles

**Overview:**  
After pagination completes, all comment items are deduplicated by author profile URL. The workflow attaches the ICP selections to each item, then loops through profiles and calls Apify to enrich each profile. A “preserve criteria” node ensures `_formData` survives the enrichment response shape.

**Nodes involved:**
- **Sticky Note - Section 3**
- **Sticky Note - Warning**
- **Remove Duplicate Commenters**
- **Attach ICP Criteria to Profiles**
- **Loop Through Profiles**
- **Enrich LinkedIn Profile**
- **Preserve ICP Criteria**

#### Node: Sticky Note - Section 3
- **Type / role:** Sticky Note (documents Block 3)
- **Connections:** None.

#### Node: Sticky Note - Warning
- **Type / role:** Sticky Note (cost warning)
- **Content:** “Apify charges per API call. Each profile enrichment is a separate call. Monitor your Apify usage…”
- **Applies conceptually to:** the enrichment loop (especially **Enrich LinkedIn Profile**).
- **Connections:** None.

#### Node: Remove Duplicate Commenters
- **Type / role:** `code` (deduplication)
- **Configuration choices:**
  - Iterates over all comment items.
  - Extracts `item.json?.author?.profile_url`.
  - Keeps first occurrence per unique `profile_url`.
  - Outputs simplified items: `{ name, profile_url }`
- **Input connections:** From **Loop Through Pages** (done output)
- **Output connections:** → **Attach ICP Criteria to Profiles**
- **Potential failures / edge cases:**
  - If Apify comment item schema differs (no `author.profile_url`), results may be empty.
  - Some commenters may have missing profile URLs → excluded.

#### Node: Attach ICP Criteria to Profiles
- **Type / role:** `code` (propagates form selections into each profile item)
- **Configuration choices:**
  - Reads the form trigger output using: `$('ICP Lead Extractor Form').first().json`
  - Handles two possible shapes:
    - If `formTrigger.data` exists, use it; else use root object.
  - Extracts:
    - `selectedTitles = formData['Target Job Titles'] || []`
    - `selectedCountries = formData['Target Countries'] || []`
  - Attaches `_formData: { selectedTitles, selectedCountries }` to each item.
- **Input:** Deduplicated commenter items
- **Output connections:** → **Loop Through Profiles**
- **Potential failures / edge cases:**
  - If the Form Trigger node name changes, `$('ICP Lead Extractor Form')` will break.
  - If the form fields are renamed, lookups like `formData['Target Job Titles']` will fail and become empty arrays.

#### Node: Loop Through Profiles
- **Type / role:** `splitInBatches` (iterates over unique commenter profiles)
- **Configuration choices:**
  - Two outputs:
    - Output 1 → **Enrich LinkedIn Profile** (loop branch)
    - Output 0 → **Filter by ICP Criteria** (done branch)
- **Input:** Items with `profile_url` and `_formData`
- **Edge cases:**
  - Enrichment is per-profile; large comment volumes can be slow/expensive.
  - Consider setting batch size to 1 explicitly to control rate.

#### Node: Enrich LinkedIn Profile
- **Type / role:** `httpRequest` (Apify profile enrichment)
- **Configuration choices:**
  - POST to Apify actor:  
    `apimaestro~linkedin-profile-detail/run-sync-get-dataset-items`
  - Timeout 10 minutes
  - JSON body:
    - `username`: `{{$json.profile_url}}` (despite name “username”, passing a URL)
    - `includeEmail`: `false`
  - Auth: `apifyApi`
- **Input:** A single profile item (recommended)
- **Output:** Enriched profile dataset item(s)
- **Output connections:** → **Preserve ICP Criteria**
- **Potential failures / edge cases:**
  - Cost per call; rate limits.
  - Actor may expect a username/handle rather than a full URL (depends on actor behavior).
  - Output shape may be an array; n8n will create items accordingly.

#### Node: Preserve ICP Criteria
- **Type / role:** `code` (runOnceForEachItem; merges enrichment output with stored `_formData`)
- **Configuration choices:**
  - `mode: runOnceForEachItem`
  - Reads original loop item via: `$('Loop Through Profiles').item.json`
  - Outputs enriched profile plus `_formData` from original
- **Input:** Enriched profile item(s)
- **Output connections:** → **Loop Through Profiles** (continues iteration)
- **Potential failures / edge cases:**
  - If node name “Loop Through Profiles” changes, `$('Loop Through Profiles')` breaks.
  - If enrichment returns multiple items per input profile, `_formData` will be duplicated across them (usually fine).

---

### 2.4 Filter & Export Leads

**Overview:**  
Once enrichment loop completes, all enriched profiles are filtered by expanded title/country synonym mappings. Matching profiles are aggregated, flattened, and converted into a CSV file returned to the form response.

**Nodes involved:**
- **Sticky Note - Section 4**
- **Filter by ICP Criteria**
- **Aggregate Matched Leads**
- **Format Leads for CSV**
- **Export to CSV**

#### Node: Sticky Note - Section 4
- **Type / role:** Sticky Note (documents Block 4)
- **Connections:** None.

#### Node: Filter by ICP Criteria
- **Type / role:** `code` (ICP matching logic)
- **Configuration choices:**
  - Expects `_formData` to exist on incoming items (added earlier and preserved).
  - Builds synonym expansions:
    - **titleMapping** for roles (CEO/Founder/VP/etc.)
    - **countryMapping** for localized names (USA/UK/Deutschland/etc.)
  - Extracts profile data from:
    - `profile.basic_info` (name, headline, location, etc.)
    - `profile.experience` (titles and locations)
  - Matching rules:
    - **Title match:** any expanded title substring matches any of:
      - up to first 3 `experience[i].title`
      - `basic_info.headline`
    - **Country match:** expanded country substring matches combined location string built from:
      - `basic_info.location` fields
      - up to first 3 `experience[i].location`
    - Must satisfy **both** title and country
  - Output: only matched profiles, with a normalized lead object.
- **Input connections:** From **Loop Through Profiles** (done output)
- **Output connections:** → **Aggregate Matched Leads**
- **Version notes:** code node typeVersion `2`.
- **Potential failures / edge cases:**
  - If items are empty, `items[0]` is undefined and the node throws “Form data not found in items”.
  - If `_formData` was not preserved, it throws (by design).
  - Substring matching can yield false positives (e.g., “CTO” matching inside unrelated text); consider word-boundary checks.
  - Country inference from free-text locations can be noisy.

#### Node: Aggregate Matched Leads
- **Type / role:** `aggregate` (collect matched leads into a single array field)
- **Configuration choices:**
  - `aggregate: aggregateAllItemData`
  - `destinationFieldName: leads`
  - Produces one item containing `leads: [ ... ]`
- **Input:** Matched leads (multiple items)
- **Output:** Single aggregated item
- **Output connections:** → **Format Leads for CSV**
- **Edge cases:**
  - If no matched items arrive, behavior depends on n8n Aggregate node version/settings; the next node handles empty `leads` defensively.

#### Node: Format Leads for CSV
- **Type / role:** `code` (flatten data into CSV columns)
- **Configuration choices:**
  - Reads `const leads = $json.leads || []`
  - If empty: returns `{ message: 'No matching profiles found' }` (not a CSV)
  - Else creates one item per lead with consistent headers:
    - Full Name, LinkedIn Profile URL, Job Title, All Job Titles, Location, Matched Job Title, Matched Country, Headline, Current Company, Email
- **Input:** Aggregated `leads` item
- **Output:** CSV-ready items
- **Output connections:** → **Export to CSV**
- **Edge cases:**
  - If no leads, downstream “Convert to File” will produce a file from the “message” object (not a typical leads CSV). If you prefer returning an error or a blank CSV, adjust logic.

#### Node: Export to CSV
- **Type / role:** `convertToFile` (generates downloadable CSV file)
- **Configuration choices:**
  - Converts incoming items to a file (CSV inferred by content/usage).
  - Filename expression:  
    `linkedin_icp_leads_{{ $now.toFormat('yyyy-MM-dd_HHmm') }}.csv`
- **Input:** Flattened lead rows
- **Output:** Binary file output (returned to the form as last node response)
- **Version notes:** typeVersion `1.1`
- **Potential failures / edge cases:**
  - If items have inconsistent fields, CSV columns may vary.
  - Large datasets can increase memory usage.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / workflow description |  |  | ## LinkedIn ICP Lead Extractor\n\nThis workflow extracts qualified leads from LinkedIn post comments based on your Ideal Customer Profile (ICP) job titles and countries.\n\n### How it works\n1. Submit a LinkedIn post URL via the form\n2. Select target job titles and countries\n3. Comments are scraped and deduplicated\n4. Each profile is enriched with detailed data\n5. Profiles are filtered against your ICP criteria\n6. Matching leads are exported as CSV\n\n### Setup steps\n- [ ] Create an Apify account at apify.com\n- [ ] Add your Apify API credentials in n8n\n- [ ] Click \"Execute Workflow\" to open the form\n- [ ] Submit a LinkedIn post URL with your ICP criteria\n\n### Requirements\n- Apify account with API access (Free tier is available on Apify)\n\n### Customization\n- Modify job titles/countries in the Form Trigger\n- Adjust pagination limits for larger posts\n- Connect output to your CRM instead of CSV |
| Sticky Note - Section 1 | n8n-nodes-base.stickyNote | Documentation (Block 1) |  |  | ## 1. User Input Form\nCollects the LinkedIn post URL and ICP filtering criteria (job titles and target countries) from the user. |
| ICP Lead Extractor Form | n8n-nodes-base.formTrigger | Collect LinkedIn URL + ICP selections |  | Prepare Pagination Requests | ## 1. User Input Form\nCollects the LinkedIn post URL and ICP filtering criteria (job titles and target countries) from the user. |
| Sticky Note - Section 2 | n8n-nodes-base.stickyNote | Documentation (Block 2) |  |  | ## 2. Scrape Comments with Pagination\nFetches all comments from the LinkedIn post using Apify. Handles pagination automatically to retrieve up to 1,000 comments. |
| Prepare Pagination Requests | n8n-nodes-base.code | Build page request items for comment scraping | ICP Lead Extractor Form | Loop Through Pages | ## 2. Scrape Comments with Pagination\nFetches all comments from the LinkedIn post using Apify. Handles pagination automatically to retrieve up to 1,000 comments. |
| Loop Through Pages | n8n-nodes-base.splitInBatches | Iterate through comment pages | Prepare Pagination Requests; Check for More Pages | Scrape LinkedIn Comments; Remove Duplicate Commenters | ## 2. Scrape Comments with Pagination\nFetches all comments from the LinkedIn post using Apify. Handles pagination automatically to retrieve up to 1,000 comments. |
| Scrape LinkedIn Comments | n8n-nodes-base.httpRequest | Apify actor call to scrape comments for a given page | Loop Through Pages | Check for More Pages | ## 2. Scrape Comments with Pagination\nFetches all comments from the LinkedIn post using Apify. Handles pagination automatically to retrieve up to 1,000 comments. |
| Check for More Pages | n8n-nodes-base.code | Return items and route back into paging loop | Scrape LinkedIn Comments | Loop Through Pages | ## 2. Scrape Comments with Pagination\nFetches all comments from the LinkedIn post using Apify. Handles pagination automatically to retrieve up to 1,000 comments. |
| Sticky Note - Section 3 | n8n-nodes-base.stickyNote | Documentation (Block 3) |  |  | ## 3. Deduplicate & Enrich Profiles\nRemoves duplicate commenters and enriches each unique profile with detailed LinkedIn data including job titles, location, and company information. |
| Remove Duplicate Commenters | n8n-nodes-base.code | Deduplicate commenters by LinkedIn profile URL | Loop Through Pages | Attach ICP Criteria to Profiles | ## 3. Deduplicate & Enrich Profiles\nRemoves duplicate commenters and enriches each unique profile with detailed LinkedIn data including job titles, location, and company information. |
| Attach ICP Criteria to Profiles | n8n-nodes-base.code | Attach selected titles/countries to each profile item | Remove Duplicate Commenters | Loop Through Profiles | ## 3. Deduplicate & Enrich Profiles\nRemoves duplicate commenters and enriches each unique profile with detailed LinkedIn data including job titles, location, and company information. |
| Loop Through Profiles | n8n-nodes-base.splitInBatches | Iterate through unique profiles for enrichment | Attach ICP Criteria to Profiles; Preserve ICP Criteria | Enrich LinkedIn Profile; Filter by ICP Criteria | ## 3. Deduplicate & Enrich Profiles\nRemoves duplicate commenters and enriches each unique profile with detailed LinkedIn data including job titles, location, and company information.\n\n### ⚠️ API Costs\nApify charges per API call. Each profile enrichment is a separate call. Monitor your Apify usage to manage costs effectively. |
| Enrich LinkedIn Profile | n8n-nodes-base.httpRequest | Apify actor call to enrich profile details | Loop Through Profiles | Preserve ICP Criteria | ## 3. Deduplicate & Enrich Profiles\nRemoves duplicate commenters and enriches each unique profile with detailed LinkedIn data including job titles, location, and company information.\n\n### ⚠️ API Costs\nApify charges per API call. Each profile enrichment is a separate call. Monitor your Apify usage to manage costs effectively. |
| Preserve ICP Criteria | n8n-nodes-base.code | Re-attach `_formData` after enrichment response | Enrich LinkedIn Profile | Loop Through Profiles | ## 3. Deduplicate & Enrich Profiles\nRemoves duplicate commenters and enriches each unique profile with detailed LinkedIn data including job titles, location, and company information. |
| Sticky Note - Section 4 | n8n-nodes-base.stickyNote | Documentation (Block 4) |  |  | ## 4. Filter & Export Leads\nFilters enriched profiles against your ICP criteria and exports matching leads as a downloadable CSV file. |
| Filter by ICP Criteria | n8n-nodes-base.code | Match enriched profiles to ICP (title + country) | Loop Through Profiles | Aggregate Matched Leads | ## 4. Filter & Export Leads\nFilters enriched profiles against your ICP criteria and exports matching leads as a downloadable CSV file. |
| Aggregate Matched Leads | n8n-nodes-base.aggregate | Aggregate all matched leads into one item | Filter by ICP Criteria | Format Leads for CSV | ## 4. Filter & Export Leads\nFilters enriched profiles against your ICP criteria and exports matching leads as a downloadable CSV file. |
| Format Leads for CSV | n8n-nodes-base.code | Flatten lead objects into CSV columns | Aggregate Matched Leads | Export to CSV | ## 4. Filter & Export Leads\nFilters enriched profiles against your ICP criteria and exports matching leads as a downloadable CSV file. |
| Export to CSV | n8n-nodes-base.convertToFile | Convert rows into downloadable CSV file | Format Leads for CSV |  | ## 4. Filter & Export Leads\nFilters enriched profiles against your ICP criteria and exports matching leads as a downloadable CSV file. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“Extract Target ICP Leads from LinkedIn Comments using Apify”** (or similar).

2) **Add the Form Trigger**
- Add node: **Form Trigger**
- Name: **ICP Lead Extractor Form**
- Set:
  - Form Title: *LinkedIn ICP Lead Extractor*
  - Form Description: a short description of what it does
  - Button label: *Extract Leads*
  - Response Mode: **Last Node**
- Add fields:
  - **LinkedIn Post URL** (Text, Required)
  - **Target Job Titles** (Checkbox, Required) with options: CEO, Founder, Co-Founder, CMO, COO, CTO, CFO, CRO, CPO, VP Marketing, VP Sales, VP Engineering, Head of Marketing, Head of Sales, Head of Growth
  - **Target Countries** (Checkbox, Required) with options: United States, United Kingdom, Germany, France, Spain, Netherlands, Canada, Australia, Ireland, Singapore

3) **Configure Apify credentials in n8n**
- Create credential: **Apify API**
- Paste your Apify token.
- Ensure it is selectable by HTTP Request nodes using **predefinedCredentialType → apifyApi**.

4) **Add pagination preparation**
- Add node: **Code**
- Name: **Prepare Pagination Requests**
- Code logic:
  - Read `$json['LinkedIn Post URL']`
  - Create items for pages 1..10 (configurable)
  - Output items containing `{ postUrl, pageNumber, _originalFormData }`
- Connect: **ICP Lead Extractor Form → Prepare Pagination Requests**

5) **Add page loop**
- Add node: **Split In Batches**
- Name: **Loop Through Pages**
- Use defaults (optionally set Batch Size to **1** for predictable per-page execution).
- Connect: **Prepare Pagination Requests → Loop Through Pages**

6) **Add comment scraping via Apify**
- Add node: **HTTP Request**
- Name: **Scrape LinkedIn Comments**
- Set:
  - Method: **POST**
  - URL: `https://api.apify.com/v2/acts/apimaestro~linkedin-post-comments-replies-engagements-scraper-no-cookies/run-sync-get-dataset-items`
  - Authentication: **Predefined Credential Type** → **Apify API**
  - Body type: **JSON**
  - JSON body fields:
    - `postIds`: `[{{$json.postUrl}}]`
    - `page_number`: `{{$json.pageNumber}}`
    - `sortOrder`: `"most recent"`
    - `limit`: `100`
  - Timeout: **600000**
- Connect loop branch:
  - **Loop Through Pages (output 1) → Scrape LinkedIn Comments**

7) **Return to loop controller**
- Add node: **Code**
- Name: **Check for More Pages**
- Keep as a pass-through returning items (as in workflow), or enhance it later.
- Connect: **Scrape LinkedIn Comments → Check for More Pages**
- Connect: **Check for More Pages → Loop Through Pages** (this creates the loop)

8) **Deduplicate commenters once paging completes**
- Add node: **Code**
- Name: **Remove Duplicate Commenters**
- Logic: iterate all comment items; keep unique `author.profile_url`; output `{ name, profile_url }`.
- Connect done branch:
  - **Loop Through Pages (output 0) → Remove Duplicate Commenters**

9) **Attach ICP selections to each profile**
- Add node: **Code**
- Name: **Attach ICP Criteria to Profiles**
- Logic:
  - Read `$('ICP Lead Extractor Form').first().json`
  - Use `.data` if present
  - Extract `Target Job Titles` + `Target Countries`
  - Attach `_formData: { selectedTitles, selectedCountries }` to each profile item
- Connect: **Remove Duplicate Commenters → Attach ICP Criteria to Profiles**

10) **Add profile enrichment loop**
- Add node: **Split In Batches**
- Name: **Loop Through Profiles**
- Optionally set Batch Size = **1** (recommended for cost/rate control).
- Connect: **Attach ICP Criteria to Profiles → Loop Through Profiles**

11) **Enrich each profile via Apify**
- Add node: **HTTP Request**
- Name: **Enrich LinkedIn Profile**
- Set:
  - Method: POST
  - URL: `https://api.apify.com/v2/acts/apimaestro~linkedin-profile-detail/run-sync-get-dataset-items`
  - Authentication: Apify API credential
  - JSON body:
    - `username`: `{{$json.profile_url}}`
    - `includeEmail`: `false`
  - Timeout: 600000
- Connect loop branch:
  - **Loop Through Profiles (output 1) → Enrich LinkedIn Profile**

12) **Preserve ICP criteria through enrichment output**
- Add node: **Code**
- Name: **Preserve ICP Criteria**
- Set mode: **Run Once for Each Item**
- Logic:
  - `const originalData = $('Loop Through Profiles').item.json;`
  - Return enriched `$json` plus `originalData._formData`
- Connect:
  - **Enrich LinkedIn Profile → Preserve ICP Criteria**
  - **Preserve ICP Criteria → Loop Through Profiles** (continues loop)

13) **Filter after enrichment loop completes**
- Add node: **Code**
- Name: **Filter by ICP Criteria**
- Connect done branch:
  - **Loop Through Profiles (output 0) → Filter by ICP Criteria**
- In code:
  - Read `_formData.selectedTitles` and `_formData.selectedCountries`
  - Expand synonyms with mappings
  - Extract titles/headline and location fields
  - Keep profiles where **titleMatch && countryMatch**
  - Output normalized lead object

14) **Aggregate matched leads**
- Add node: **Aggregate**
- Name: **Aggregate Matched Leads**
- Set:
  - Aggregate: **Aggregate All Item Data**
  - Destination field: **leads**
- Connect: **Filter by ICP Criteria → Aggregate Matched Leads**

15) **Format for CSV**
- Add node: **Code**
- Name: **Format Leads for CSV**
- Flatten `leads` array into row items with consistent column names.
- Connect: **Aggregate Matched Leads → Format Leads for CSV**

16) **Export CSV**
- Add node: **Convert to File**
- Name: **Export to CSV**
- Set file name expression:  
  `linkedin_icp_leads_{{$now.toFormat('yyyy-MM-dd_HHmm')}}.csv`
- Connect: **Format Leads for CSV → Export to CSV**
- Ensure this is the **last node** so the form returns the file.

17) **(Optional) Add Sticky Notes**
- Add Sticky Notes to label the 4 sections and the cost warning, matching your team’s conventions.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create an Apify account and add Apify API credentials in n8n. | Mentioned in the main sticky note. |
| Apify charges per API call; each profile enrichment is a separate call. Monitor usage to manage costs. | Cost warning sticky note (applies mainly to profile enrichment loop). |
| Customization ideas: adjust job titles/countries in the form; adjust pagination limits; connect output to CRM instead of CSV. | Mentioned in the main sticky note. |
| Apify (account creation) | https://apify.com |

