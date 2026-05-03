Extract leads from Google Maps into Google Sheets using the Places API

https://n8nworkflows.xyz/workflows/extract-leads-from-google-maps-into-google-sheets-using-the-places-api-13753


# Extract leads from Google Maps into Google Sheets using the Places API

# Extract leads from Google Maps into Google Sheets using the Places API

## 1. Workflow Overview

This workflow collects business leads from Google Maps via the Google Places API (New) and stores them in Google Sheets. It supports two execution modes:

- **Manual / on-demand:** a user submits a search query through an n8n form
- **Scheduled / batch:** the workflow periodically reads unscripted queries from a Google Sheet and processes them automatically

Its main purpose is lead generation and enrichment for local businesses, such as searching for “Bakeries New York USA” or “Coworking spaces Berlin Germany”, then saving structured business details like phone number, address, website, map URL, ratings, and derived links.

### 1.1 Entry Points
The workflow begins from one of two triggers:
- a **Form Trigger** for single ad hoc searches
- a **Schedule Trigger** for bulk processing from a Google Sheet

### 1.2 Query Retrieval and Validation
If the scheduled path is used, the workflow reads pending search terms from the **SEARCH QUERIES** sheet and filters out empty entries before forwarding them into the common processing path.

### 1.3 Search Execution Against Google Places API
Both entry paths converge, then the workflow sends the query to Google Places API `v1/places:searchText`, with pagination enabled to collect multiple result pages.

### 1.4 Result Expansion and Field Mapping
The API response contains an array of places. The workflow splits that array into one item per place and maps raw Google fields into a normalized lead structure.

### 1.5 Lead Storage and Deduplication
Each mapped place is written into the **LEADS** sheet using an append-or-update operation keyed by the Google Place ID, preventing duplicate lead rows.

### 1.6 Scheduled Query Completion Tracking
If the query came from the scheduled sheet-based path, the workflow marks that source query row as scraped so it is not processed again.

---

## 2. Block-by-Block Analysis

## 2.1 Entry Points

**Overview:**  
This block defines the two possible ways to start the workflow. One path is interactive through an n8n form, and the other is automated through a time-based schedule.

**Nodes Involved:**  
- Scrape on form submission
- Scrape on schedule

### Node: Scrape on form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Receives a user-submitted form request and starts the workflow with form data.
- **Configuration choices:**  
  - Form title: **Scrape Google Maps leads**
  - Description: asks for a Google Maps-style business query
  - One required field: **Search Query**
  - Placeholder example: `Coworking spaces Berlin Germany`
- **Key expressions or variables used:**  
  - Produces `Search Query` in the incoming item JSON
- **Input and output connections:**  
  - No input; this is a trigger
  - Output goes to **Merge** input 0
- **Version-specific requirements:**  
  - Uses form trigger typeVersion `2.4`
- **Edge cases or potential failure types:**  
  - Form is unusable if the workflow is inactive or webhook/form URL is not reachable
  - Missing required field blocks submission
- **Sub-workflow reference:**  
  - None

### Node: Scrape on schedule
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Launches the workflow on a repeating schedule.
- **Configuration choices:**  
  - Interval-based schedule
  - Runs every hour
- **Key expressions or variables used:**  
  - None
- **Input and output connections:**  
  - No input; this is a trigger
  - Output goes to **Get rows in QUERIES**
- **Version-specific requirements:**  
  - Uses schedule trigger typeVersion `1.3`
- **Edge cases or potential failure types:**  
  - Workflow must be active for schedule execution
  - Timezone differences can affect expected runtime perception
- **Sub-workflow reference:**  
  - None

---

## 2.2 Scheduled Query Retrieval and Validation

**Overview:**  
This block is only used by the scheduled execution path. It fetches rows from the query source sheet where the `Scraped` flag is empty or false, then removes rows with empty search text.

**Nodes Involved:**  
- Get rows in QUERIES
- Discard empty queries

### Node: Get rows in QUERIES
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads pending search queries from a Google Sheet.
- **Configuration choices:**  
  - Operation: read rows with filters
  - Spreadsheet: `Leads from Google Maps`
  - Sheet: `SEARCH QUERIES`
  - Filters combined with **OR**
  - Filter 1: `Scraped = false`
  - Filter 2: `Scraped` empty
  - `returnFirstMatch: true`
- **Key expressions or variables used:**  
  - Later nodes reference:
    - `$('Get rows in QUERIES').item.json.row_number`
    - `$('Get rows in QUERIES').params.sheetName.value`
    - `$('Get rows in QUERIES').params.documentId.value`
- **Input and output connections:**  
  - Input from **Scrape on schedule**
  - Output to **Discard empty queries**
- **Version-specific requirements:**  
  - Uses Google Sheets node typeVersion `4.7`
  - Requires valid Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**  
  - OAuth errors or expired Google credentials
  - Wrong sheet/document selection
  - If `returnFirstMatch` is enabled, only one pending query is processed per run
  - If the `Scraped` column contains values other than actual empty/false representations, rows may be missed
- **Sub-workflow reference:**  
  - None

### Node: Discard empty queries
- **Type and technical role:** `n8n-nodes-base.filter`  
  Ensures only rows with a non-empty `Search Query` move forward.
- **Configuration choices:**  
  - Condition: `Search Query` is not empty
  - Strict type validation
- **Key expressions or variables used:**  
  - `={{ $json["Search Query"] }}`
- **Input and output connections:**  
  - Input from **Get rows in QUERIES**
  - Output to **Merge** input 1
- **Version-specific requirements:**  
  - Uses filter node typeVersion `2.3`
- **Edge cases or potential failure types:**  
  - If the sheet column name changes, the expression will fail logically
  - Rows with whitespace-only strings may still need manual normalization depending on actual source values
- **Sub-workflow reference:**  
  - None

---

## 2.3 Unified Query Routing

**Overview:**  
This block merges the manual and scheduled query paths so the downstream Google Places search logic only needs one input path.

**Nodes Involved:**  
- Merge

### Node: Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines incoming items from the form path and the scheduled sheet path into a single downstream stream.
- **Configuration choices:**  
  - No explicit mode overrides are set; it acts as a routing convergence point
- **Key expressions or variables used:**  
  - None
- **Input and output connections:**  
  - Input 0 from **Scrape on form submission**
  - Input 1 from **Discard empty queries**
  - Output to **Google Places Search**
- **Version-specific requirements:**  
  - Uses merge node typeVersion `3.2`
- **Edge cases or potential failure types:**  
  - If both triggers produce data simultaneously, behavior depends on execution context and item flow timing
  - Upstream item structure must include `Search Query`
- **Sub-workflow reference:**  
  - None

---

## 2.4 Google Places Search Execution

**Overview:**  
This block sends the search text to the Google Places API (New), using pagination to fetch additional pages of results when available.

**Nodes Involved:**  
- Google Places Search

### Node: Google Places Search
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs a POST request to the Google Places API text search endpoint.
- **Configuration choices:**  
  - URL: `https://places.googleapis.com/v1/places:searchText`
  - Method: `POST`
  - Sends query parameters and headers
  - Query parameter:
    - `textQuery = {{ $json["Search Query"] }}`
  - Headers:
    - `X-Goog-Api-Key = YourGoogePlacesAPIKeyHere`
    - `X-Goog-FieldMask = *`
    - `Content-Type = application/json`
  - Pagination enabled:
    - body parameter `pageToken = {{ $response.body.nextPageToken }}`
    - max requests: `3`
    - request interval: `500 ms`
    - stop when there is no `nextPageToken`
- **Key expressions or variables used:**  
  - `={{ $json["Search Query"] }}`
  - `={{ $response.body.nextPageToken }}`
  - `={{ !$response.body.nextPageToken }}`
- **Input and output connections:**  
  - Input from **Merge**
  - Output to **Split Out Places**
- **Version-specific requirements:**  
  - Uses HTTP Request node typeVersion `4.3`
- **Edge cases or potential failure types:**  
  - Invalid or restricted API key
  - Places API not enabled in Google Cloud
  - Billing disabled or quota exceeded
  - Pagination token timing issues if Google requires token warm-up delay longer than 500 ms
  - Using `X-Goog-FieldMask: *` may increase payload size and cost
  - Query sent as URL query parameter rather than a JSON body field; if API behavior changes, this may need adjustment
- **Sub-workflow reference:**  
  - None

---

## 2.5 Result Expansion and Lead Field Mapping

**Overview:**  
This block transforms the Google Places API response into one item per place and creates a normalized lead schema suitable for sheet storage.

**Nodes Involved:**  
- Split Out Places
- Map Places Fields

### Node: Split Out Places
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the `places` array from the API response into separate items.
- **Configuration choices:**  
  - Field to split: `places`
- **Key expressions or variables used:**  
  - None
- **Input and output connections:**  
  - Input from **Google Places Search**
  - Output to **Map Places Fields**
- **Version-specific requirements:**  
  - Uses splitOut node typeVersion `1`
- **Edge cases or potential failure types:**  
  - If `places` is missing or empty, no lead items will be produced
  - API error payloads may not contain `places`, resulting in silent no-output depending on node behavior
- **Sub-workflow reference:**  
  - None

### Node: Map Places Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a cleaned and normalized field set from each place result.
- **Configuration choices:**  
  - Adds structured lead fields such as:
    - `source_id`
    - `name`
    - `address`
    - `phone`
    - `social_searcher`
    - `website`
    - `website_analysis_url`
    - `google_maps_url`
    - `rating`
    - `reviews`
    - `coordinates`
    - `address_components`
- **Key expressions or variables used:**  
  - `source_id`:
    - `{{ $json?.id || $json?.name?.startsWith("places/") ? $json?.name?.split("/")?.[1] : null}}`
    - Intended to derive the Google Place ID
  - `name`:
    - `{{ $json?.displayName?.text || $json?.name || null}}`
  - `address`:
    - `{{ $json?.formattedAddress || $json?.shortFormattedAddress || null}}`
  - `phone`:
    - `{{ $json?.internationalPhoneNumber?.replaceAll("+","00")?.replaceAll(" ", "") || null}}`
  - `social_searcher`:
    - URL built from encoded display name
  - `website_analysis_url`:
    - Pagespeed Web URL built from encoded website
  - `google_maps_url`:
    - direct Google Maps URI if available, otherwise generated from place ID
  - `coordinates`:
    - `"latitude,longitude"`
  - `address_components`:
    - JSON-like string assembled from route, locality, state, postal code, country, country code, neighborhood
- **Input and output connections:**  
  - Input from **Split Out Places**
  - Output to **Append or update lead row**
- **Version-specific requirements:**  
  - Uses Set node typeVersion `3.4`
- **Edge cases or potential failure types:**  
  - The `source_id` expression may suffer from operator precedence ambiguity; if `$json.id` exists, validate the resulting value carefully
  - `google_maps_url` references `$json?.places?.googleMapsUri`, but after splitting, each item is usually a single place object, so this path may be unnecessary or incorrect; fallback generation is what likely matters
  - `address_components` is stored as a string, not an actual object
  - Missing optional Google fields produce null values
- **Sub-workflow reference:**  
  - None

---

## 2.6 Lead Storage and Deduplication

**Overview:**  
This block writes each lead into the destination sheet and prevents duplicates by matching on the unique Google Place ID.

**Nodes Involved:**  
- Append or update lead row

### Node: Append or update lead row
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Stores mapped leads in Google Sheets with upsert logic.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Spreadsheet: `Leads from Google Maps`
  - Sheet: `LEADS`
  - Matching column: `source_id`
  - Auto-map style configuration with explicit column expressions
  - Fields written include:
    - `source_id`
    - `name`
    - `category`
    - `address`
    - `phone`
    - `social_searcher`
    - `website`
    - `website_analysis_url`
    - `google_maps_url`
    - `rating`
    - `reviews`
    - `coordinates`
    - `address_components`
  - Type conversion disabled
- **Key expressions or variables used:**  
  - Similar expressions to the Set node, but directly from raw place JSON rather than mapped output
  - `category`:
    - `{{ $json?.primaryTypeDisplayName?.text || $json?.primaryType || $json?.types?.[0] || null }}`
- **Input and output connections:**  
  - Input from **Map Places Fields**
  - Output to **If**
- **Version-specific requirements:**  
  - Uses Google Sheets node typeVersion `4.7`
  - Requires Google Sheets OAuth2 credentials
  - Target sheet must contain matching column names
- **Edge cases or potential failure types:**  
  - Credential failures or spreadsheet access permission errors
  - If sheet headers do not match configured schema, writes may fail or map incorrectly
  - The node ignores the mapped fields from the Set node for many columns and recomputes them from raw input; future edits must keep both nodes aligned
  - Upsert relies on `source_id`; if the expression is wrong or null, duplicate protection breaks
- **Sub-workflow reference:**  
  - None

---

## 2.7 Scheduled Completion Marking

**Overview:**  
This block checks whether the execution came from the scheduled Google Sheets path. If yes, it marks the processed query row as scraped in the source sheet.

**Nodes Involved:**  
- If
- Mark Query as Scraped

### Node: If
- **Type and technical role:** `n8n-nodes-base.if`  
  Detects whether **Get rows in QUERIES** was executed during the current run.
- **Configuration choices:**  
  - Boolean condition: `$('Get rows in QUERIES').isExecuted` is true
- **Key expressions or variables used:**  
  - `={{$('Get rows in QUERIES').isExecuted}}`
- **Input and output connections:**  
  - Input from **Append or update lead row**
  - True output to **Mark Query as Scraped**
  - False output unused
- **Version-specific requirements:**  
  - Uses If node typeVersion `2.3`
- **Edge cases or potential failure types:**  
  - Because this node runs after every processed place, it may evaluate true for each item in a scheduled execution
  - True branch behavior is moderated by the downstream node’s `executeOnce` setting
- **Sub-workflow reference:**  
  - None

### Node: Mark Query as Scraped
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the source query row to indicate it has been processed.
- **Configuration choices:**  
  - Operation: `update`
  - `executeOnce: true`
  - Dynamic document and sheet values are taken from **Get rows in QUERIES**
  - Matching column: `row_number`
  - Sets:
    - `Scraped = TRUE`
    - `row_number = {{ $('Get rows in QUERIES').item.json.row_number }}`
- **Key expressions or variables used:**  
  - `={{ $('Get rows in QUERIES').item.json.row_number }}`
  - `={{ $('Get rows in QUERIES').params.sheetName.value }}`
  - `={{ $('Get rows in QUERIES').params.documentId.value }}`
- **Input and output connections:**  
  - Input from **If**
  - No downstream output
- **Version-specific requirements:**  
  - Uses Google Sheets node typeVersion `4.7`
  - Requires same Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**  
  - If `row_number` is missing, the update will fail or affect no row
  - If the scheduled read path was changed, these dynamic references can break
  - `executeOnce` is important; without it, the same query row could be updated redundantly for every returned place
- **Sub-workflow reference:**  
  - None

---

## 2.8 Documentation / In-Canvas Notes

**Overview:**  
These nodes are non-executable notes embedded in the workflow canvas. They document setup, entry points, API key guidance, and sheet behavior.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note5

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation.
- **Configuration choices:**  
  - Describes search and extraction setup
  - Includes spreadsheet copy link
  - Notes max pages impact on cost
- **Input and output connections:**  
  - None
- **Edge cases or potential failure types:**  
  - None
- **Sub-workflow reference:**  
  - None

### Node: Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  - Explains API key placement in `X-Goog-Api-Key`
  - Recommends Google Cloud quotas and environment-variable storage
- **Input and output connections:**  
  - None
- **Edge cases or potential failure types:**  
  - None
- **Sub-workflow reference:**  
  - None

### Node: Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  - Describes the two entry points
- **Input and output connections:**  
  - None
- **Edge cases or potential failure types:**  
  - None
- **Sub-workflow reference:**  
  - None

### Node: Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  - Describes data storage and deduplication area
- **Input and output connections:**  
  - None
- **Edge cases or potential failure types:**  
  - None
- **Sub-workflow reference:**  
  - None

### Node: Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  - Large overview note containing workflow purpose, setup steps, copy-sheet link, and security reminder
- **Input and output connections:**  
  - None
- **Edge cases or potential failure types:**  
  - None
- **Sub-workflow reference:**  
  - None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scrape on form submission | Form Trigger | Manual entry point for a single search query |  | Merge | ## 1. Entry Points<br>Choose between the Form node for on-demand searches or the Schedule trigger to process a bulk list of queries from your Google Sheet.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Scrape on schedule | Schedule Trigger | Scheduled entry point for batch search processing |  | Get rows in QUERIES | ## 1. Entry Points<br>Choose between the Form node for on-demand searches or the Schedule trigger to process a bulk list of queries from your Google Sheet.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Get rows in QUERIES | Google Sheets | Reads one pending query row from the SEARCH QUERIES sheet | Scrape on schedule | Discard empty queries | ## 2. Search & Extraction<br>This section fetches data from the Places API (New).<br>1.Setup Credentials<br>2. Copy [this](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy?usp=sharing) spreadsheet<br>3. Add queries for the scheduled execution in the QUERIES  (i.e. "Bakeries New York USA", "Cafes Berlin Germany" etc.).<br>Leave the **Scraped** column empty or set to 'false'<br><br>Note: Adjust "Max Pages" in the HTTP node to control result volume and API costs.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Discard empty queries | Filter | Removes rows without a usable Search Query value | Get rows in QUERIES | Merge | ## 2. Search & Extraction<br>This section fetches data from the Places API (New).<br>1.Setup Credentials<br>2. Copy [this](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy?usp=sharing) spreadsheet<br>3. Add queries for the scheduled execution in the QUERIES  (i.e. "Bakeries New York USA", "Cafes Berlin Germany" etc.).<br>Leave the **Scraped** column empty or set to 'false'<br><br>Note: Adjust "Max Pages" in the HTTP node to control result volume and API costs.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Merge | Merge | Converges manual and scheduled query inputs | Scrape on form submission; Discard empty queries | Google Places Search | ## 2. Search & Extraction<br>This section fetches data from the Places API (New).<br>1.Setup Credentials<br>2. Copy [this](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy?usp=sharing) spreadsheet<br>3. Add queries for the scheduled execution in the QUERIES  (i.e. "Bakeries New York USA", "Cafes Berlin Germany" etc.).<br>Leave the **Scraped** column empty or set to 'false'<br><br>Note: Adjust "Max Pages" in the HTTP node to control result volume and API costs.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Google Places Search | HTTP Request | Calls Google Places Text Search API with pagination | Merge | Split Out Places | ## 2. Search & Extraction<br>This section fetches data from the Places API (New).<br>1.Setup Credentials<br>2. Copy [this](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy?usp=sharing) spreadsheet<br>3. Add queries for the scheduled execution in the QUERIES  (i.e. "Bakeries New York USA", "Cafes Berlin Germany" etc.).<br>Leave the **Scraped** column empty or set to 'false'<br><br>Note: Adjust "Max Pages" in the HTTP node to control result volume and API costs.<br><br>## 2. API Key and Max Pages<br><br>1. Provide your Google Places (new) API Key<br>in the **X-Goog-Api-Key** body parameter value<br><br>- This API key must be generated from a Google Cloud Project with the "Places API" enabled.<br><br>### Optional configurations:<br><br>- Adjust the **Max Pages** to control how many pages of results are fetched for each query. (Mind that more pages means more requests to the Google API and potentially more costs. Googles allows you to easily set alerts and strict quotas per api endpoint to prevent unexpected charges. This particular node is using the **v1/places:searchText** endpoint.<br><br>- API Key Security: The API key is currently hardcoded directly into the "Google Places Search" HTTP Request node. For production and business critical accounts, it is strongly advised to secure this key using an n8n environment variable to prevent exposure and facilitate easier management AND limit the key access to the v1/places:searchText API request only<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Split Out Places | Split Out | Expands the API response into one item per place | Google Places Search | Map Places Fields | ## 2. Search & Extraction<br>This section fetches data from the Places API (New).<br>1.Setup Credentials<br>2. Copy [this](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy?usp=sharing) spreadsheet<br>3. Add queries for the scheduled execution in the QUERIES  (i.e. "Bakeries New York USA", "Cafes Berlin Germany" etc.).<br>Leave the **Scraped** column empty or set to 'false'<br><br>Note: Adjust "Max Pages" in the HTTP node to control result volume and API costs.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Map Places Fields | Set | Normalizes Google Places fields into lead-friendly fields | Split Out Places | Append or update lead row | ## 2. Search & Extraction<br>This section fetches data from the Places API (New).<br>1.Setup Credentials<br>2. Copy [this](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy?usp=sharing) spreadsheet<br>3. Add queries for the scheduled execution in the QUERIES  (i.e. "Bakeries New York USA", "Cafes Berlin Germany" etc.).<br>Leave the **Scraped** column empty or set to 'false'<br><br>Note: Adjust "Max Pages" in the HTTP node to control result volume and API costs.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Append or update lead row | Google Sheets | Upserts leads into the LEADS sheet using source_id for deduplication | Map Places Fields | If | ##  Data Storage<br>Nodes here handle lead deduplication and write new results to your "LEADS" sheet. If the workflow was scheduled, it updates the source query status to prevent duplicates.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| If | If | Detects whether the scheduled Google Sheets path was used | Append or update lead row | Mark Query as Scraped | ##  Data Storage<br>Nodes here handle lead deduplication and write new results to your "LEADS" sheet. If the workflow was scheduled, it updates the source query status to prevent duplicates.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Mark Query as Scraped | Google Sheets | Updates the source query row to set Scraped=TRUE | If |  | ##  Data Storage<br>Nodes here handle lead deduplication and write new results to your "LEADS" sheet. If the workflow was scheduled, it updates the source query status to prevent duplicates.<br><br># Google Maps Lead Extractor [Places API (New)]<br><br>## How it works<br>This workflow automates lead collection from Google Maps using two entry points:<br>* **Manual:** Submit a single query (e.g., "Cafes in Paris") via the n8n Form.<br>* **Automated:** A Schedule trigger fetches pending queries from a Google Sheet.<br><br>The workflow calls the **Places API (New)**, parses business details (name, phone, website, Social Searcher and Pagespeed links), and saves them to your destination sheet. It includes a deduplication step using the **Google Place ID** to ensure you don't save the same lead twice. If running on a schedule, it automatically marks queries as "scraped" in your source sheet.<br><br>## Setup steps<br>1. **API Key:** Enable the **Places API (New)** in Google Cloud Console and paste your key into the 'Google Places Search' node, `X-Goog-Api-Key` parameter value.<br>2. **Sheet Template:** Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy).<br>3. **Credentials:** Connect your Google Sheets OAuth2 account to all Google Sheets nodes.<br>4. **Configure Nodes:** Ensure the 'Get rows in QUERIES' and 'Append or update lead row' nodes are pointing to your new spreadsheet copy.<br>5. **Security:** For production, move your API key from the HTTP header to an n8n Secret or Environment Variable. |
| Sticky Note | Sticky Note | Canvas documentation for search/extraction setup |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for API key and pagination guidance |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for entry points |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for data storage behavior |  |  |  |
| Sticky Note5 | Sticky Note | Canvas documentation for full workflow setup |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   **Extract leads from Google Maps into Google Sheets using the Places API**

2. **Create the manual trigger**
   - Add a **Form Trigger** node
   - Name it **Scrape on form submission**
   - Set:
     - Form title: `Scrape Google Maps leads`
     - Form description: `Enter a Google Maps Query to find business leads`
   - Add one required field:
     - Label: `What businesses are you looking for and where`
     - Field name: `Search Query`
     - Placeholder: `Coworking spaces Berlin Germany`

3. **Create the scheduled trigger**
   - Add a **Schedule Trigger** node
   - Name it **Scrape on schedule**
   - Set it to run every **1 hour**

4. **Prepare Google Sheets credentials**
   - In n8n, create or connect a **Google Sheets OAuth2** credential
   - Ensure it has access to the spreadsheet you will use

5. **Prepare the spreadsheet structure**
   - Make a copy of this spreadsheet template:  
     [Google Sheet copy link](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy)
   - Ensure you have at least:
     - a source sheet for queries, named similar to **SEARCH QUERIES**
     - a destination sheet for results, named **LEADS**
   - In the query sheet, include:
     - `Search Query`
     - `Scraped`
     - `row_number` available through Google Sheets node metadata
   - In the LEADS sheet, include columns:
     - `source_id`
     - `name`
     - `category`
     - `address`
     - `phone`
     - `social_searcher`
     - `website`
     - `website_analysis_url`
     - `google_maps_url`
     - `rating`
     - `reviews`
     - `coordinates`
     - `address_components`

6. **Add the query-reading Google Sheets node**
   - Add a **Google Sheets** node
   - Name it **Get rows in QUERIES**
   - Select your Google Sheets credential
   - Choose your copied spreadsheet
   - Choose the source query sheet
   - Configure it to read rows using filters:
     - `Scraped = false`
     - OR `Scraped` is empty
   - Set `Combine Filters` to **OR**
   - Enable `Return First Match`
   - Connect:
     - **Scrape on schedule** → **Get rows in QUERIES**

7. **Add a filter to remove blank queries**
   - Add a **Filter** node
   - Name it **Discard empty queries**
   - Create condition:
     - left value: `{{ $json["Search Query"] }}`
     - operator: **is not empty**
   - Connect:
     - **Get rows in QUERIES** → **Discard empty queries**

8. **Add a merge node**
   - Add a **Merge** node
   - Name it **Merge**
   - Keep default merge behavior
   - Connect:
     - **Scrape on form submission** → **Merge** input 1
     - **Discard empty queries** → **Merge** input 2

9. **Create the Google Places API key**
   - In Google Cloud Console:
     - create/select a project
     - enable **Places API (New)**
     - enable billing
     - create an API key
   - Prefer restricting the key to:
     - the Places API endpoint(s) you need
     - approved source/IP restrictions where possible

10. **Add the HTTP Request node for Google Places search**
    - Add an **HTTP Request** node
    - Name it **Google Places Search**
    - Set:
      - Method: `POST`
      - URL: `https://places.googleapis.com/v1/places:searchText`
    - Enable sending query parameters
    - Add query parameter:
      - `textQuery` = `{{ $json["Search Query"] }}`
    - Enable sending headers
    - Add headers:
      - `X-Goog-Api-Key` = your API key
      - `X-Goog-FieldMask` = `*`
      - `Content-Type` = `application/json`
    - Configure pagination:
      - pass `pageToken` in the request body using:
        - `{{ $response.body.nextPageToken }}`
      - max requests: `3`
      - request interval: `500`
      - stop when:
        - `{{ !$response.body.nextPageToken }}`
    - Connect:
      - **Merge** → **Google Places Search**

11. **Add the array splitter**
    - Add a **Split Out** node
    - Name it **Split Out Places**
    - Field to split out: `places`
    - Connect:
      - **Google Places Search** → **Split Out Places**

12. **Add the field-mapping Set node**
    - Add a **Set** node
    - Name it **Map Places Fields**
    - Add these fields as expressions:
      - `source_id`
      - `name`
      - `address`
      - `phone`
      - `social_searcher`
      - `website`
      - `website_analysis_url`
      - `google_maps_url`
      - `rating`
      - `reviews`
      - `coordinates`
      - `address_components`
    - Use logic equivalent to:
      - place ID from `id` or `name`
      - display name from `displayName.text`
      - address from formatted address fields
      - phone normalized with `+` converted to `00`
      - website-derived links for Social Searcher and PageSpeed
      - coordinates from latitude and longitude
      - address parts extracted from `addressComponents`
    - Connect:
      - **Split Out Places** → **Map Places Fields**

13. **Add the destination Google Sheets upsert node**
    - Add another **Google Sheets** node
    - Name it **Append or update lead row**
    - Select the same Google Sheets credential
    - Set operation to **Append or Update**
    - Choose your spreadsheet
    - Choose the **LEADS** sheet
    - Set matching column to:
      - `source_id`
    - Map the output fields to the sheet columns:
      - `source_id`
      - `name`
      - `category`
      - `address`
      - `phone`
      - `social_searcher`
      - `website`
      - `website_analysis_url`
      - `google_maps_url`
      - `rating`
      - `reviews`
      - `coordinates`
      - `address_components`
    - Important:
      - The original workflow recomputes many values directly from the raw place payload instead of using only the Set node outputs
      - For a cleaner rebuild, you may choose to map directly from **Map Places Fields** outputs, but if you want exact parity, mirror the original expressions
    - Connect:
      - **Map Places Fields** → **Append or update lead row**

14. **Add a conditional node to detect scheduled runs**
    - Add an **If** node
    - Name it **If**
    - Configure condition:
      - expression: `{{ $('Get rows in QUERIES').isExecuted }}`
      - evaluate as boolean true
    - Connect:
      - **Append or update lead row** → **If**

15. **Add the query status update node**
    - Add another **Google Sheets** node
    - Name it **Mark Query as Scraped**
    - Select the same Google Sheets credential
    - Set operation to **Update**
    - Enable **Execute Once**
    - Set document ID dynamically from **Get rows in QUERIES**
    - Set sheet name dynamically from **Get rows in QUERIES**
    - Use matching column:
      - `row_number`
    - Map values:
      - `Scraped` = `TRUE`
      - `row_number` = `{{ $('Get rows in QUERIES').item.json.row_number }}`
    - Connect:
      - **If** true output → **Mark Query as Scraped**

16. **Add optional sticky notes**
    - Add sticky notes for:
      - entry point explanation
      - API key setup
      - sheet copy/setup
      - lead storage and deduplication
    - This is optional for execution but useful for maintenance

17. **Test the form path**
    - Submit a query such as:
      - `Cafes Berlin Germany`
    - Verify:
      - API returns places
      - rows are written into **LEADS**
      - `source_id` is populated
      - no source query row is updated, since form runs should not mark scheduled rows

18. **Test the schedule path**
    - Add a row in **SEARCH QUERIES**
      - `Search Query = Bakeries New York USA`
      - `Scraped = false` or blank
    - Run the scheduled path manually once
    - Verify:
      - one pending query is read
      - places are written/upserted into **LEADS**
      - the source query row gets `Scraped = TRUE`

19. **Harden the workflow for production**
    - Replace the hardcoded API key with:
      - an n8n environment variable, or
      - an n8n secret
    - Consider replacing `X-Goog-FieldMask = *` with only required fields to reduce payload size and cost
    - Consider increasing the pagination interval if Google rejects fresh page tokens
    - Validate `source_id` extraction carefully

20. **Recommended implementation refinement**
    - Use the mapped fields from **Map Places Fields** consistently in the Google Sheets write node
    - Simplify the `source_id` expression with explicit parentheses
    - If you want to process more than one queued search per schedule run, disable `Return First Match` or redesign batching behavior

### Sub-workflow setup
This workflow does **not** use any Execute Workflow / sub-workflow nodes.  
There are **no sub-workflows to configure**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Copy the spreadsheet template before using the workflow | [https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy](https://docs.google.com/spreadsheets/d/1x_GYw6KHgvhe2dgSw1eKKTimIzQYTsRkkWoxkAJEdD8/copy) |
| Scheduled execution expects search queries in the source sheet, with `Scraped` left empty or set to `false` | Source sheet operational note |
| The workflow uses Google Places API (New), specifically `v1/places:searchText` | Google Cloud / Places API configuration |
| Adjusting the HTTP pagination max pages directly affects result volume, request count, and Google API cost | HTTP Request node behavior |
| The API key is hardcoded in the HTTP node in the original workflow; for production use, move it to an n8n secret or environment variable | Security recommendation |
| Restrict the API key and consider setting quotas/alerts in Google Cloud to prevent unexpected billing | Google Cloud operational safeguard |
| The workflow deduplicates leads through the Google Place ID stored in `source_id` | LEADS sheet upsert logic |