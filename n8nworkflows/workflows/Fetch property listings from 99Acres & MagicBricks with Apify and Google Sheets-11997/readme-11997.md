Fetch property listings from 99Acres & MagicBricks with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/fetch-property-listings-from-99acres---magicbricks-with-apify-and-google-sheets-11997


# Fetch property listings from 99Acres & MagicBricks with Apify and Google Sheets

## 1. Workflow Overview

**Purpose:** Collect property listings from **99Acres** and **MagicBricks** using **Apify scrapers**, normalize the fields (ID, title, price, price/sqft, URL), remove duplicates, and write results into a **newly created Google Spreadsheet** with two dedicated sheets.

**Target use cases:**
- Quickly exporting search results from Indian real-estate portals into a structured spreadsheet
- Running repeatable scraping runs driven by user-provided search URLs
- Keeping portal results separated while using a unified schema

### 1.1 Input Reception (Form submission)
User submits two search URLs via an n8n Form Trigger.

### 1.2 Spreadsheet Provisioning
A new Google Spreadsheet is created dynamically, with separate tabs for each portal.

### 1.3 External Data Collection (Apify)
Two HTTP requests call Apify “actor” endpoints to scrape listings based on the submitted URLs.

### 1.4 Data Cleaning & Standardization
Two Code nodes deduplicate and map raw listing fields into a common schema.

### 1.5 Persistence (Google Sheets Append)
Each portal’s standardized rows are appended to its respective sheet tab in the newly created spreadsheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Form submission)
**Overview:** Starts the workflow when a user submits the 99Acres and MagicBricks search URLs through an n8n-hosted form.  
**Nodes involved:** `Property Search URLs`

#### Node: Property Search URLs
- **Type / role:** `Form Trigger` — workflow entrypoint via form submission (webhook-backed).
- **Configuration (interpreted):**
  - Form title: **“Real Estate Links”**
  - Fields:
    - **99Acres** (URL pasted by user)
    - **Magic Bricks** (URL pasted by user)
  - Generates a webhook endpoint (internally identified by `webhookId`) that receives submitted values.
- **Key expressions / variables:** Output is later read via expressions like:
  - `{{ $('Property Search URLs').item.json['99Acres'] }}`
  - `{{ $('Property Search URLs').item.json['Magic Bricks'] }}`
- **Connections:**
  - **Output →** `Create Master Spreadsheet`
- **Edge cases / failure modes:**
  - Missing/blank fields: downstream HTTP nodes will send invalid Apify input and may return empty datasets or errors.
  - User pastes non-search URLs or unsupported pages: Apify actor may fail or return no items.
  - Form Trigger availability depends on n8n instance URL and active workflow state.

---

### Block 2 — Spreadsheet Provisioning
**Overview:** Creates a brand-new spreadsheet per run, and pre-creates two sheets (“99Acres”, “MagicBricks”). Downstream nodes reference the created spreadsheet and sheet IDs dynamically.  
**Nodes involved:** `Create Master Spreadsheet`

#### Node: Create Master Spreadsheet
- **Type / role:** `Google Sheets` — creates a spreadsheet resource.
- **Configuration (interpreted):**
  - **Resource:** Spreadsheet
  - **Operation:** Create (via “title” + initial sheets definition)
  - **Title:** `={{ $now }}` (timestamp-based name)
  - **Sheets created:**
    - Sheet 0: **99Acres**
    - Sheet 1: **MagicBricks**
- **Key expressions / variables used later:**
  - Spreadsheet ID: `$('Create Master Spreadsheet').item.json.spreadsheetId`
  - Sheet IDs:
    - `$('Create Master Spreadsheet').item.json.sheets[0].properties.sheetId`
    - `$('Create Master Spreadsheet').item.json.sheets[1].properties.sheetId`
- **Connections:**
  - **Input ←** `Property Search URLs`
  - **Output →** `Fetch 99Acres Listings` and `Fetch MagicBricks Listings` (parallel branches)
- **Version-specific notes:** Uses Google Sheets node **typeVersion 4.7**; UI/options differ across major versions.
- **Edge cases / failure modes:**
  - Google OAuth credential not configured/expired → auth failure.
  - Google API quota limits / rate limiting.
  - Spreadsheet creation succeeds but later append fails if returned sheet structure differs (rare, but can happen if Google API response changes or permissions restrict sheet creation).

---

### Block 3 — External Data Collection (Apify)
**Overview:** Calls two Apify actor endpoints (one for 99Acres, one for MagicBricks) using the URLs submitted in the form, returning raw listing items.  
**Nodes involved:** `Fetch 99Acres Listings`, `Fetch MagicBricks Listings`

#### Node: Fetch 99Acres Listings
- **Type / role:** `HTTP Request` — triggers Apify actor run and returns dataset items synchronously.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.apify.com/v2/acts/fatihtahta~99acres-scraper-ppe/run-sync-get-dataset-items?token=<YOUR-API-KEY>`
    - Requires replacing `<YOUR-API-KEY>` with a real Apify token (or moving it to credentials/environment variables).
  - **Body:** JSON with `startUrls` list containing the submitted 99Acres URL:
    - Uses: `{{ $('Property Search URLs').item.json['99Acres'] }}`
- **Connections:**
  - **Input ←** `Create Master Spreadsheet`
  - **Output →** `Filter & Format 99Acres Listings`
- **Edge cases / failure modes:**
  - Invalid/missing Apify token → 401/403.
  - Actor not available, renamed, or insufficient plan limits → 404/402-like Apify errors.
  - Scraper blocked / captcha / portal layout changes → empty results or actor failure.
  - Large result sets → long execution time; may hit n8n timeout depending on instance settings.

#### Node: Fetch MagicBricks Listings
- **Type / role:** `HTTP Request` — triggers Apify actor run and returns dataset items synchronously.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.apify.com/v2/acts/ecomscrape~magicbricks-property-search-scraper/run-sync-get-dataset-items?token=<YOUR-API-KEY>`
  - **Body:** JSON including:
    - `max_retries_per_url: 2`
    - `proxy` settings:
      - `useApifyProxy: true`
      - `apifyProxyGroups: ["RESIDENTIAL"]`
      - `apifyProxyCountry: "IN"`
    - `urls` list containing the submitted MagicBricks URL:
      - `{{ $('Property Search URLs').item.json['Magic Bricks'] }}`
- **Connections:**
  - **Input ←** `Create Master Spreadsheet`
  - **Output →** `Filter & Format MagicBricks Listings`
- **Edge cases / failure modes:**
  - Same token/actor availability risks as above.
  - Proxy group/country mismatch with your Apify account access → run failure.
  - MagicBricks anti-bot measures may still reduce success even with proxy.

---

### Block 4 — Data Cleaning & Standardization
**Overview:** Deduplicates items by `id` and maps portal-specific fields into a shared schema to match Google Sheets columns.  
**Nodes involved:** `Filter & Format 99Acres Listings`, `Filter & Format MagicBricks Listings`

#### Node: Filter & Format 99Acres Listings
- **Type / role:** `Code` — transforms incoming items (JS).
- **Configuration (interpreted):**
  - Creates an in-node `seenIds` Set and iterates over `$input.all()`
  - Drops duplicates where `data.id` repeats
  - Outputs standardized fields:
    - `Property_id` ← `data.id`
    - `Property_title` ← `data.title`
    - `Property_price` ← `data.priceRange`
    - `Property_pricePerSqft` ← `data.pricePerSqft`
    - `Property_url` ← `data.url`
- **Connections:**
  - **Input ←** `Fetch 99Acres Listings`
  - **Output →** `Append Rows → 99Acres Sheet`
- **Edge cases / failure modes:**
  - If Apify output does not include `id` (or it’s null), deduplication won’t work as intended (all items may appear unique or collapse incorrectly if `id` is constant).
  - Missing fields yield `undefined` cells in Sheets.
  - This removes duplicates **within the single run payload** only; it does not prevent duplicates across multiple workflow executions.

#### Node: Filter & Format MagicBricks Listings
- **Type / role:** `Code` — transforms incoming items (JS).
- **Configuration (interpreted):**
  - Deduplicates by `data.id`
  - Builds a full URL:
    - If `data.url` exists: `https://www.magicbricks.com/propertyDetails/${data.url}`
    - Else: `null`
  - Outputs standardized fields:
    - `Property_id` ← `data.id`
    - `Property_title` ← `data.name`
    - `Property_price` ← template `${data.currency}${data.price}`
    - `Property_pricePerSqft` ← `data.price_per_sq_ft`
    - `Property_url` ← `fullUrl`
- **Connections:**
  - **Input ←** `Fetch MagicBricks Listings`
  - **Output →** `Append Rows → MagicBricks Sheet`
- **Edge cases / failure modes:**
  - If `currency` or `price` is missing, `Property_price` becomes strings like `undefinedundefined`.
  - If `data.url` is already a full URL, this logic will produce a malformed URL (double prefixing). It assumes `data.url` is a path/slug.
  - Same “dedupe only within one run” limitation as above.

---

### Block 5 — Persistence (Append to Google Sheets)
**Overview:** Appends standardized rows into the correct sheet tab using IDs returned by the spreadsheet creation node.  
**Nodes involved:** `Append Rows → 99Acres Sheet`, `Append Rows → MagicBricks Sheet`

#### Node: Append Rows → 99Acres Sheet
- **Type / role:** `Google Sheets` — append rows to a sheet.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Document:** dynamic `documentId` from created spreadsheet:
    - `={{ $('Create Master Spreadsheet').item.json.spreadsheetId }}`
  - **Sheet:** dynamic `sheetName` provided as sheet ID (sheet 0):
    - `={{ $('Create Master Spreadsheet').item.json.sheets[0].properties.sheetId }}`
  - **Columns schema defined:** `Property_id`, `Property_title`, `Property_price`, `Property_pricePerSqft`, `Property_url`
  - **Credentials:** Google Sheets OAuth2
- **Connections:**
  - **Input ←** `Filter & Format 99Acres Listings`
- **Edge cases / failure modes:**
  - Auth/permission issues writing to Drive.
  - If sheet ID expression fails (e.g., spreadsheet creation output missing), append will fail.
  - If no input items (empty listing result), append may write nothing (or in some configurations may error depending on n8n version).

#### Node: Append Rows → MagicBricks Sheet
- **Type / role:** `Google Sheets` — append rows to a sheet.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Document:** `={{ $('Create Master Spreadsheet').item.json.spreadsheetId }}`
  - **Sheet:** `={{ $('Create Master Spreadsheet').item.json.sheets[1].properties.sheetId }}` (sheet 1)
  - Same standardized column schema as above
  - **Credentials:** Google Sheets OAuth2
- **Connections:**
  - **Input ←** `Filter & Format MagicBricks Listings`
- **Edge cases / failure modes:** same as 99Acres append node.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Property Search URLs | Form Trigger | Collects portal search URLs; starts workflow | — | Create Master Spreadsheet | ##  99Acres & MagicBricks → Google Sheets; ### How it works …; ### Setup steps …; No manual sheet or spreadsheet selection is required. |
| Property Search URLs | Form Trigger | Collects portal search URLs; starts workflow | — | Create Master Spreadsheet | ## Trigger; Starts the workflow when a user submits property search URLs. |
| Create Master Spreadsheet | Google Sheets | Creates a new spreadsheet with two tabs | Property Search URLs | Fetch 99Acres Listings; Fetch MagicBricks Listings | ##  99Acres & MagicBricks → Google Sheets; ### How it works …; ### Setup steps …; No manual sheet or spreadsheet selection is required. |
| Create Master Spreadsheet | Google Sheets | Creates a new spreadsheet with two tabs | Property Search URLs | Fetch 99Acres Listings; Fetch MagicBricks Listings | ## Create Spreadsheet; Creates a new Google Spreadsheet with dedicated sheets for each portal. |
| Fetch 99Acres Listings | HTTP Request | Calls Apify actor to fetch 99Acres listings | Create Master Spreadsheet | Filter & Format 99Acres Listings | ## Fetch Property Listings; Calls external APIFY scrapers/APIs to retrieve raw property listings from 99Acres and MagicBricks based on user-provided URLs. |
| Fetch MagicBricks Listings | HTTP Request | Calls Apify actor to fetch MagicBricks listings | Create Master Spreadsheet | Filter & Format MagicBricks Listings | ## Fetch Property Listings; Calls external APIFY scrapers/APIs to retrieve raw property listings from 99Acres and MagicBricks based on user-provided URLs. |
| Filter & Format 99Acres Listings | Code | Deduplicates + maps 99Acres fields to unified schema | Fetch 99Acres Listings | Append Rows → 99Acres Sheet | ## Clean & Standardize Data; Filters duplicate listings and converts raw API responses into a consistent format suitable for Google Sheets. |
| Filter & Format MagicBricks Listings | Code | Deduplicates + maps MagicBricks fields to unified schema | Fetch MagicBricks Listings | Append Rows → MagicBricks Sheet | ## Clean & Standardize Data; Filters duplicate listings and converts raw API responses into a consistent format suitable for Google Sheets. |
| Append Rows → 99Acres Sheet | Google Sheets | Appends standardized rows into “99Acres” tab | Filter & Format 99Acres Listings | — | ## Save Results to Sheets; Appends cleaned listings into the correct sheet so each portal’s data stays separated and organized. |
| Append Rows → MagicBricks Sheet | Google Sheets | Appends standardized rows into “MagicBricks” tab | Filter & Format MagicBricks Listings | — | ## Save Results to Sheets; Appends cleaned listings into the correct sheet so each portal’s data stays separated and organized. |
| Sticky Note | Sticky Note | Documentation / canvas note | — | — | ##  99Acres & MagicBricks → Google Sheets; ### How it works …; ### Setup steps …; No manual sheet or spreadsheet selection is required. |
| Sticky Note1 | Sticky Note | Documentation / canvas note | — | — | ## Trigger; Starts the workflow when a user submits property search URLs. |
| Sticky Note2 | Sticky Note | Documentation / canvas note | — | — | ## Create Spreadsheet; Creates a new Google Spreadsheet with dedicated sheets for each portal. |
| Sticky Note3 | Sticky Note | Documentation / canvas note | — | — | ## Fetch Property Listings; Calls external APIFY scrapers/APIs to retrieve raw property listings from 99Acres and MagicBricks based on user-provided URLs. |
| Sticky Note4 | Sticky Note | Documentation / canvas note | — | — | ## Clean & Standardize Data; Filters duplicate listings and converts raw API responses into a consistent format suitable for Google Sheets. |
| Sticky Note5 | Sticky Note | Documentation / canvas note | — | — | ## Save Results to Sheets; Appends cleaned listings into the correct sheet so each portal’s data stays separated and organized. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it (e.g.) **“Real Estet Automation”** (original spelling) or your preferred name.

2. **Add trigger: Form Trigger**
   - Node type: **Form Trigger**
   - Form title: **Real Estate Links**
   - Add 2 fields:
     1) Label: **99Acres**, placeholder: “Enter Url of 99acres search Page”  
     2) Label: **Magic Bricks**, placeholder: “Enter Url of Magic Bricks search Page”
   - This node becomes the workflow entrypoint.

3. **Add Google Sheets node: Create Master Spreadsheet**
   - Node type: **Google Sheets**
   - **Resource:** Spreadsheet
   - **Operation:** Create spreadsheet
   - **Title:** set expression to `{{$now}}`
   - In the “Sheets” UI (initial sheet definitions), add:
     - Sheet name **99Acres**
     - Sheet name **MagicBricks**
   - **Credentials:** configure **Google Sheets OAuth2** (sign in; ensure it can create spreadsheets in your Drive).
   - Connect: **Form Trigger → Create Master Spreadsheet**

4. **Add HTTP Request node: Fetch 99Acres Listings**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL:  
     `https://api.apify.com/v2/acts/fatihtahta~99acres-scraper-ppe/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`
   - Body type: **JSON**
   - JSON body (with expression):
     - `startUrls` as an array containing: `{{ $('Property Search URLs').item.json['99Acres'] }}`
   - Connect: **Create Master Spreadsheet → Fetch 99Acres Listings**

5. **Add HTTP Request node: Fetch MagicBricks Listings**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL:  
     `https://api.apify.com/v2/acts/ecomscrape~magicbricks-property-search-scraper/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`
   - Body type: **JSON**
   - JSON body includes:
     - `max_retries_per_url: 2`
     - `proxy.useApifyProxy: true`
     - `proxy.apifyProxyGroups: ["RESIDENTIAL"]`
     - `proxy.apifyProxyCountry: "IN"`
     - `urls: [ {{ $('Property Search URLs').item.json['Magic Bricks'] }} ]`
   - Connect: **Create Master Spreadsheet → Fetch MagicBricks Listings**

6. **Add Code node: Filter & Format 99Acres Listings**
   - Node type: **Code**
   - Paste logic to:
     - dedupe by `id`
     - output fields: `Property_id`, `Property_title`, `Property_price`, `Property_pricePerSqft`, `Property_url`
   - Connect: **Fetch 99Acres Listings → Filter & Format 99Acres Listings**

7. **Add Code node: Filter & Format MagicBricks Listings**
   - Node type: **Code**
   - Paste logic to:
     - dedupe by `id`
     - build `Property_url` as `https://www.magicbricks.com/propertyDetails/${data.url}`
     - output unified fields
   - Connect: **Fetch MagicBricks Listings → Filter & Format MagicBricks Listings**

8. **Add Google Sheets node: Append Rows → 99Acres Sheet**
   - Node type: **Google Sheets**
   - Operation: **Append**
   - **Document ID (expression):** `{{ $('Create Master Spreadsheet').item.json.spreadsheetId }}`
   - **Sheet (by ID, expression):** `{{ $('Create Master Spreadsheet').item.json.sheets[0].properties.sheetId }}`
   - Define columns (mapping) to match the Code output:
     - Property_id, Property_title, Property_price, Property_pricePerSqft, Property_url
   - Credentials: same Google Sheets OAuth2
   - Connect: **Filter & Format 99Acres Listings → Append Rows → 99Acres Sheet**

9. **Add Google Sheets node: Append Rows → MagicBricks Sheet**
   - Same configuration as above, but:
     - **Sheet ID expression:** `{{ $('Create Master Spreadsheet').item.json.sheets[1].properties.sheetId }}`
   - Connect: **Filter & Format MagicBricks Listings → Append Rows → MagicBricks Sheet**

10. **Credentials & secrets**
   - **Google Sheets OAuth2:** required on the 3 Google Sheets nodes.
   - **Apify token:** replace `<YOUR-API-KEY>` in both HTTP Request URLs.
     - Recommended: store token in an n8n credential, environment variable, or workflow static data; avoid hardcoding.

11. **Test run**
   - Open the Form Trigger test mode (or activate workflow and submit live).
   - Submit valid search URLs for both portals.
   - Verify a new spreadsheet is created and filled with rows in both tabs.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “99Acres & MagicBricks → Google Sheets” workflow note explaining flow, setup steps, and that no manual spreadsheet selection is required. | Embedded as Sticky Note content on the canvas. |
| Disclaimer (French): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Provided in the request; applies to the overall content handling and compliance context. |