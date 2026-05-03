Track Amazon prices and competitor offers with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/track-amazon-prices-and-competitor-offers-with-apify-and-google-sheets-12788


# Track Amazon prices and competitor offers with Apify and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow automatically tracks Amazon product prices (your listing vs a competitor/other seller) using **Apify’s Amazon Product Scraper**, then writes the latest pricing back into **Google Sheets** on a daily schedule.

**Typical use cases:**
- Amazon sellers monitoring their own listing and a competitor listing for each product/SKU
- Growth/ops teams maintaining a pricing sheet updated daily without manual checks
- Scalable monitoring by adding more rows (products) and extending the scraper input/mapping

### 1.1 Daily Trigger
Runs every day at a specified hour.

### 1.2 Load Product Rows from Google Sheets
Reads the sheet containing product metadata and the two Amazon URLs to compare.

### 1.3 Row-by-Row Processing Loop
Iterates through each row and, per row, runs the scraping + update sequence.

### 1.4 Scrape Amazon via Apify + Fetch Dataset Output
Runs an Apify actor for the current row, then retrieves the structured results from the actor’s dataset.

### 1.5 Normalize/Extract Prices (JavaScript)
Selects prices from the actor output and merges them with the original row context (category/material + row number + timestamp).

### 1.6 Write Back to Google Sheets + Continue Loop
Appends or updates the matching sheet row (by “Material Name”), then loops to the next input row.

---

## 2. Block-by-Block Analysis

### Block 1 — Daily Trigger
**Overview:** Starts the workflow automatically once per day at the configured time.

**Nodes involved:**
- Schedule Trigger

**Node details**
- **Schedule Trigger** (n8n-nodes-base.scheduleTrigger, v1.2)  
  - **Role:** Time-based entry point.
  - **Configuration:** Runs daily at **08:00** (server/workflow timezone context; n8n instance timezone matters).
  - **Outputs to:** `Get row(s) in sheet`
  - **Potential failures / edge cases:**
    - Timezone mismatch (expected local 8 AM vs server 8 AM).
    - If n8n is stopped at trigger time, execution won’t occur unless catch-up is implemented externally.

---

### Block 2 — Fetch product row data from Sheet
**Overview:** Loads all rows from the configured Google Sheet which contain category/material identifiers and Amazon URLs.

**Nodes involved:**
- Get row(s) in sheet

**Node details**
- **Get row(s) in sheet** (n8n-nodes-base.googleSheets, v4.7)  
  - **Role:** Read input dataset (products to monitor).
  - **Configuration choices (interpreted):**
    - Uses a specific spreadsheet (document ID `1hr4e9y0z2...`) and sheet tab `Sheet1` (`gid=0`).
    - Operation is “get rows” (default behavior implied by node name and minimal params).
  - **Credentials:** Google Sheets OAuth2 (`Google Sheets account 3`)
  - **Outputs to:** `Loop Over Items`
  - **Key fields expected in each row (based on later expressions):**
    - `Category Name`
    - `Material Name`
    - `Amazon S3P (Revlon)`
    - `Amazon P3P (RK)`
    - `row_number` (n8n Google Sheets node typically provides this; required later by the Code node)
  - **Potential failures / edge cases:**
    - OAuth expiration / insufficient permissions.
    - Sheet renamed or tab changed (Sheet1/gid mismatch).
    - Missing columns or different header names will break later expressions like `$json['Amazon S3P (Revlon)']`.
    - Empty sheet returns no items → loop block does nothing (silent “success”).

---

### Block 3 — Processing each row (loop)
**Overview:** Iterates through each row from Google Sheets, sending each row into the scraping pipeline, and repeating after sheet update.

**Nodes involved:**
- Loop Over Items

**Node details**
- **Loop Over Items** (n8n-nodes-base.splitInBatches, v3)  
  - **Role:** Batch/loop controller to process items sequentially.
  - **Configuration:** Default options; batch size not explicitly set (n8n default is typically 1 for Split in Batches unless configured—verify in UI).
  - **Connections:**
    - **Input:** From `Get row(s) in sheet`
    - **Output 1 (main index 0):** Not used (left empty in connections)
    - **Output 2 (main index 1):** To `Run an Actor` (this is the “next item” path in many SplitInBatches configurations)
    - **Loop-back:** `Append or update row in sheet` outputs back into `Loop Over Items` to continue.
  - **Potential failures / edge cases:**
    - If the batch output wiring is wrong (using the wrong output index), the loop may never run or may run only once.
    - If downstream nodes error and stop execution, remaining rows won’t be processed (some nodes here use “continueRegularOutput” to reduce stoppage).
    - Large sheets can cause long runtimes; consider execution timeouts and rate limits.

---

### Block 4 — Scrape and get Amazon data (Apify)
**Overview:** For each sheet row, runs Apify’s Amazon scraper actor on two URLs (your listing + competitor listing).

**Nodes involved:**
- Run an Actor

**Node details**
- **Run an Actor** (@apify/n8n-nodes-apify.apify, v1)  
  - **Role:** Triggers Apify actor execution.
  - **Error handling:** `onError: continueRegularOutput` (workflow continues even if this node errors; output may be partial/empty).
  - **Configuration choices:**
    - Actor: **Amazon Product Scraper (junglee/Amazon-crawler)** (`BG3WDrGdteHgZgbPK`)
    - Input payload (`customBody`) sends `categoryOrProductUrls` with two entries:
      - `url` from `{{ $json['Amazon S3P (Revlon)'] }}`
      - `url` from `{{ $json['Amazon P3P (RK)'] }}`
  - **Credentials:** Apify API (`Apify account 3`)
  - **Outputs to:** `HTTP Request1`
  - **Key expressions/variables:**
    - `$json['Amazon S3P (Revlon)']`, `$json['Amazon P3P (RK)']` are taken from the current loop item (sheet row).
  - **Potential failures / edge cases:**
    - Invalid/missing URLs → actor may fail or return empty dataset.
    - Actor requires paid Apify plan (noted in sticky note); free plan may block or throttle.
    - Amazon anti-bot / CAPTCHA can reduce data quality or cause failures.
    - If actor output schema changes (price fields, etc.), downstream Code node may produce `N/A`.
    - Because errors continue, you can silently write bad/empty pricing unless validated.

---

### Block 5 — Fetch Apify output from actor (HTTP)
**Overview:** Retrieves the actor run’s dataset items via Apify API.

**Nodes involved:**
- HTTP Request1

**Node details**
- **HTTP Request1** (n8n-nodes-base.httpRequest, v4.2)  
  - **Role:** Calls Apify dataset items endpoint to fetch scraped results.
  - **Error handling:** `onError: continueRegularOutput`
  - **Configuration choices:**
    - URL template:  
      `https://api.apify.com/v2/datasets/{{ $json.defaultDatasetId }}/items?token={your_apify_api_YOUR_TOKEN_HERE}`
    - Uses `defaultDatasetId` from the **Run an Actor** output.
  - **Outputs to:** `Code in JavaScript`
  - **Potential failures / edge cases:**
    - **Token placeholder risk:** The URL contains `{your_apify_api_YOUR_TOKEN_HERE}` which must be replaced. If not replaced, this will 401/403.
    - Better practice is using Apify credentials/auth rather than embedding token in URL (reduces leakage).
    - Dataset may not be ready immediately for very large runs; may return empty array if fetched too early (actor usually completes before returning, but depends on node behavior).
    - Rate limiting from Apify.
    - Response shape: this node likely returns multiple items (one per URL), which the next Code node assumes.

---

### Block 6 — Extract price data By Code Node
**Overview:** Converts the Apify dataset items into a single normalized record (rowNumber + two prices + metadata + timestamp).

**Nodes involved:**
- Code in JavaScript

**Node details**
- **Code in JavaScript** (n8n-nodes-base.code, v2)  
  - **Role:** Data transformation and field extraction.
  - **Error handling:** `onError: continueRegularOutput`
  - **Logic (interpreted):**
    - Reads all incoming dataset items: `const items = $input.all();`
    - Extracts:
      - `price1` from `items[0].json.price.value` or `items[0].json.listPrice.value`, else `'N/A'`
      - `price2` from `items[1]...` similarly
    - Fetches original sheet row context from the loop:  
      `const originalRow = $('Loop Over Items').item.json;`
    - Outputs **one item** with:
      - `rowNumber: originalRow.row_number`
      - `MyListingPrice: price1`
      - `OtherSellerPrice: price2`
      - `categoryName`, `materialName`
      - `lastUpdated` in `Asia/Kolkata` locale time
  - **Outputs to:** `Append or update row in sheet`
  - **Potential failures / edge cases:**
    - If Apify returned only 0–1 items, `items[1]` is undefined → code safely falls back to `'N/A'`.
    - If Apify schema differs (no `price.value` nor `listPrice.value`), output becomes `'N/A'`.
    - Reference `$('Loop Over Items').item.json` depends on n8n’s item linking; if the loop node changes behavior or the execution is parallelized, it can reference the wrong item.
    - Output field names (`MyListingPrice`, `OtherSellerPrice`) **do not match** the next Google Sheets mapping (which expects `rkPrice`, `revlonPrice`, etc.). This is a functional mismatch that will likely result in blank writes unless the sheet mapping is adjusted.

---

### Block 7 — Update data to sheet with final output
**Overview:** Writes extracted fields back into Google Sheets, updating existing rows matched by “Material Name” or appending if not found, then continues the loop.

**Nodes involved:**
- Append or update row in sheet

**Node details**
- **Append or update row in sheet** (n8n-nodes-base.googleSheets, v4.7)  
  - **Role:** Persist results back to the sheet.
  - **Operation:** `appendOrUpdate`
  - **Matching:** `matchingColumns: ["Material Name"]`  
    - Updates the row where “Material Name” matches; otherwise appends a new row.
  - **Column mapping (as configured):**
    - `Material Name` = `{{ $json.materialName }}`
    - `Revlon Price` = `{{ $json.revlonPrice }}`
    - `RK Price` = `{{ $json.rkPrice }}`
  - **Credentials:** Google Sheets OAuth2 (`Google Sheets account 3`)
  - **Connections:**
    - **Input:** From `Code in JavaScript`
    - **Output:** Back to `Loop Over Items` to continue iteration
  - **Critical integration note (field mismatch):**
    - The Code node outputs `MyListingPrice` and `OtherSellerPrice`, but this Sheets node expects `revlonPrice` and `rkPrice`. Unless other data exists, `Revlon Price` and `RK Price` will be empty.
    - Fix by either:
      - Renaming Code output fields to `revlonPrice` and `rkPrice`, or
      - Updating this node’s mapping to use `{{$json.MyListingPrice}}` and `{{$json.OtherSellerPrice}}`.
  - **Potential failures / edge cases:**
    - If “Material Name” is empty/null, matching can fail and create duplicates.
    - Type conversion disabled; numeric prices may remain strings (usually fine for Sheets but affects sorting/calcs).
    - OAuth permission issues.
    - Concurrent executions can race and overwrite updates if scheduled too frequently.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily time-based start | — | Get row(s) in sheet | ## Daily Auto Start |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Read product rows/URLs from Google Sheets | Schedule Trigger | Loop Over Items | ## Fetch product row data from Sheet |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterate rows and control loop | Get row(s) in sheet; Append or update row in sheet | Run an Actor (via output index 1) | ## Processing each row |
| Run an Actor | @apify/n8n-nodes-apify.apify | Run Apify Amazon scraper actor for two URLs | Loop Over Items | HTTP Request1 | ## Scrape and get Amazon data |
| HTTP Request1 | n8n-nodes-base.httpRequest | Fetch actor dataset items from Apify API | Run an Actor | Code in JavaScript | ## Fetch Apify output from actor |
| Code in JavaScript | n8n-nodes-base.code | Extract/normalize prices and add metadata | HTTP Request1 | Append or update row in sheet | ## Extract price data By Code Node |
| Append or update row in sheet | n8n-nodes-base.googleSheets | Update/append results into Google Sheets | Code in JavaScript | Loop Over Items | ## Update data to sheet with final output |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment | — | — |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Comment / documentation | — | — |  |

> Note: Sticky Note nodes are included as nodes (per requirement), but their content is captured primarily in Section 5 and the per-node “Sticky Note” column above applies to functional nodes they visually label.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: **“Track Amazon prices and monitor competitors”** (or the provided title variant).

2. **Add node: Schedule Trigger**  
   - Node type: **Schedule Trigger**  
   - Set schedule to **Daily** at **08:00** (or set “triggerAtHour: 8”).  
   - Connect: `Schedule Trigger` → `Get row(s) in sheet`.

3. **Add node: Google Sheets (Get row(s))**  
   - Node type: **Google Sheets**  
   - Credentials: configure **Google Sheets OAuth2** (connect a Google account with access to the spreadsheet).  
   - Operation: **Get Many / Read rows** (exact wording depends on UI version; it should output all rows).  
   - Select:
     - Document: your spreadsheet (ID like `1hr4e9y0z2...`)
     - Sheet tab: `Sheet1` (gid=0)  
   - Ensure the header row includes at least:
     - `Category Name`
     - `Material Name`
     - `Amazon S3P (Revlon)`
     - `Amazon P3P (RK)`
   - Connect: `Get row(s) in sheet` → `Loop Over Items`.

4. **Add node: Split In Batches (Loop Over Items)**  
   - Node type: **Split In Batches**  
   - Batch size: set to **1** (recommended for per-row processing; confirm in UI).  
   - Connect its **“Next Batch/Item” output (commonly output 2)** to the scraping chain:
     - `Loop Over Items` → `Run an Actor` (make sure you use the output that advances items).

5. **Add node: Apify (Run an Actor)**  
   - Node type: **Apify** → action **Run an Actor**  
   - Credentials: configure **Apify API token** in n8n credentials.  
   - Actor: select **Amazon Product Scraper (junglee/Amazon-crawler)** (or your chosen actor).  
   - Input/body: provide JSON containing the two URLs from the current row, e.g.
     - `categoryOrProductUrls[0].url` = `{{$json["Amazon S3P (Revlon)"]}}`
     - `categoryOrProductUrls[1].url` = `{{$json["Amazon P3P (RK)"]}}`
   - Error handling: optionally set **Continue on Fail** to avoid stopping the whole batch.
   - Connect: `Run an Actor` → `HTTP Request1`.

6. **Add node: HTTP Request (fetch dataset items)**  
   - Node type: **HTTP Request**  
   - Method: GET  
   - URL: `https://api.apify.com/v2/datasets/{{$json.defaultDatasetId}}/items`  
   - Authentication (recommended): use Apify token via header/query in a secure way (credentials), not a hardcoded string.  
     - If you must use query token: `...?token=YOUR_TOKEN` (but avoid embedding in workflow JSON).
   - Response: JSON  
   - Error handling: optionally **Continue on Fail**.
   - Connect: `HTTP Request1` → `Code in JavaScript`.

7. **Add node: Code (JavaScript)**  
   - Node type: **Code**  
   - Paste logic to:
     - Read `$input.all()` dataset items
     - Extract price fields (price.value / listPrice.value)
     - Pull original row from the loop node (`$('Loop Over Items').item.json`)
     - Return one combined object
   - **Important:** Align output field names with what you will write to Sheets. For example, either:
     - Output `revlonPrice` and `rkPrice`, **or**
     - Keep `MyListingPrice` and `OtherSellerPrice` and map those in the Sheets node.
   - Connect: `Code in JavaScript` → `Append or update row in sheet`.

8. **Add node: Google Sheets (Append or update row)**  
   - Node type: **Google Sheets**  
   - Credentials: same Google OAuth2  
   - Operation: **Append or Update**  
   - Document + Sheet: same as read step  
   - Matching column: **Material Name**  
   - Map output fields to columns, for example:
     - `Material Name` = `{{$json.materialName}}`
     - `Revlon Price` = `{{$json.revlonPrice}}` (or `{{$json.MyListingPrice}}`)
     - `RK Price` = `{{$json.rkPrice}}` (or `{{$json.OtherSellerPrice}}`)
     - Optionally add `Last Updated` column and map `{{$json.lastUpdated}}`
   - Connect: `Append or update row in sheet` → back to `Loop Over Items` to continue processing.

9. **(Optional but recommended) Add data validation**
   - Add an IF node before updating Sheets to avoid writing `'N/A'` or empty values.
   - Add retries/backoff on HTTP Request if Apify rate limits.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Track Amazon prices and monitor competitors” + “How It Works” + setup and enhancement ideas; scalable by extending columns/nodes. | Sticky note text embedded in the workflow canvas |
| “Note: The Amazon scraper requires a paid plan. A free trial is available.” | Apify actor usage note (cost/plan constraint) |
| “Need Help? Connect with me at faisalahmed1928@gmail.com” | Contact from sticky note |

