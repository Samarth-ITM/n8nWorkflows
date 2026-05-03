Turn closed-won HubSpot deals into lookalike prospects with CompanyEnrich

https://n8nworkflows.xyz/workflows/turn-closed-won-hubspot-deals-into-lookalike-prospects-with-companyenrich-12227


# Turn closed-won HubSpot deals into lookalike prospects with CompanyEnrich

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow runs on a weekly schedule, pulls companies from HubSpot, selects the “best” ones (top X% by annual revenue), extracts their domains, calls CompanyEnrich’s “similar companies” endpoint to generate lookalike prospects, and then appends/updates those prospects into a Google Sheet while avoiding duplicates by matching on the `domain` column.

**Typical use cases**
- Building prospect lists from your best customers.
- Enriching outbound targeting with lookalike companies.
- Maintaining a continuously updated sheet of similar companies without manual research.

### Logical blocks
**1.1 Scheduling & HubSpot data collection**
- Triggers weekly and fetches HubSpot companies.

**1.2 Best-company selection**
- Sorts companies by annual revenue and keeps only the top percent.

**1.3 Domain preparation & batching**
- Extracts a domain seed per selected company and loops item-by-item.

**1.4 CompanyEnrich lookalike enrichment**
- Sends the domain to CompanyEnrich, receives a list of similar companies, and splits the list into individual items.

**1.5 Normalization & persistence**
- Normalizes fields and writes each similar company to Google Sheets with “append-or-update” logic using `domain` as the unique key.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & HubSpot data collection

**Overview:** Starts the workflow on a weekly cadence and retrieves all HubSpot companies using a HubSpot Private App token.

**Nodes involved:**  
- Schedule Trigger  
- HubSpot Get Companies  

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entrypoint.
- **Configuration (interpreted):** Runs at an interval measured in **weeks** (default “every 1 week” unless further configured in UI).
- **Connections:** Outputs to **HubSpot Get Companies**.
- **Edge cases / failures:**
  - None typical besides disabled workflow (workflow is currently `active: false`).
  - If instance time zone differs from expectation, the effective run time may surprise you.

#### Node: HubSpot Get Companies
- **Type / role:** `n8n-nodes-base.hubspot` — HubSpot CRM data retrieval.
- **Configuration (interpreted):**
  - **Resource:** Company
  - **Operation:** Get All
  - **Return All:** true (pulls all companies, not paginated subset)
  - **Authentication:** HubSpot **App Token** (Private App token).
- **Credentials:** `hubspotAppToken`
- **Connections:** Outputs to **Filter Best**.
- **Version specifics:** Node `typeVersion: 1`.
- **Edge cases / failures:**
  - Auth/scopes: token must have at least company read scope; insufficient scopes cause 401/403.
  - Large portals: returning all companies may be slow and can hit rate limits/timeouts.
  - Properties availability: if `annualrevenue` isn’t present or not numeric, downstream ranking becomes less meaningful.

**Sticky note coverage (contextual):**  
- **“Schedule Trigger and Filter”**: adjust schedule interval; adjust `TOP_PERCENT` in Filter node code.

---

### 2.2 Best-company selection

**Overview:** Ranks HubSpot companies by annual revenue and keeps only a configurable top percentage.

**Nodes involved:**  
- Filter Best  

#### Node: Filter Best
- **Type / role:** `n8n-nodes-base.function` — custom JavaScript filtering/sorting.
- **Configuration (interpreted):**
  - Defines `TOP_PERCENT = 5;`
  - Maps inbound items to raw `company` JSON objects.
  - Sorts descending by `a.properties?.annualrevenue` (parsed as float).
  - Selects `count = max(1, floor(total * TOP_PERCENT/100))`, ensuring at least 1 company.
  - Outputs only those top companies as items.
- **Key expressions / variables:**
  - `TOP_PERCENT` (customize to 1–20% depending on dataset size).
  - Revenue parse: `parseFloat(a.properties?.annualrevenue || 0)`
- **Connections:** Receives from **HubSpot Get Companies**, outputs to **Extract Domain**.
- **Edge cases / failures:**
  - **Data shape mismatch risk:** Some HubSpot node outputs expose properties as `properties.annualrevenue` (string), while other nodes (or older exports) may provide `properties.annualrevenue.value`. This code assumes the former. If your HubSpot node returns `{ properties: { annualrevenue: { value: "..." }}}`, sorting will treat all revenue as 0.
  - Non-numeric values (e.g., “unknown”) become `NaN`, which can break stable ordering; consider normalizing `NaN` to 0.
  - If there are zero companies, output is empty; the rest of the workflow won’t run.

**Sticky note coverage (contextual):**  
- **“Schedule Trigger and Filter”**: explains adjusting interval and `TOP_PERCENT`.

---

### 2.3 Domain preparation & batching

**Overview:** Extracts a usable domain seed for each selected company and processes them in a controlled loop (batching).

**Nodes involved:**  
- Extract Domain  
- Loop Over Items  

#### Node: Extract Domain
- **Type / role:** `n8n-nodes-base.function` — transforms HubSpot company item into a minimal `{domain}` seed.
- **Configuration (interpreted):**
  - Reads `item.json.properties || {}` into `p`.
  - Chooses domain from:
    1) `p.domain?.value`  
    2) `p.website?.value`  
    3) empty string
  - Outputs `{ json: { domain } }`.
- **Connections:** Receives from **Filter Best**, outputs to **Loop Over Items**.
- **Edge cases / failures:**
  - **Property shape mismatch risk (again):** If HubSpot returns `properties.domain` as a plain string rather than `{ value }`, `p.domain?.value` will be `undefined`. Similarly for `website`. You may need to support both shapes.
  - Empty domain leads to CompanyEnrich calls with `domain: ""`, which will likely return an error or irrelevant results.

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — iteration controller to process items sequentially/in batches.
- **Configuration (interpreted):**
  - No explicit batch size shown (defaults apply in UI; commonly 1).
  - Uses the standard SplitInBatches pattern:
    - **Output 0** is the “done” path (here unused).
    - **Output 1** is the “continue/batch” path feeding the loop body.
- **Connections:**
  - Input from **Extract Domain**
  - Output (index 1) to **HTTP Request**
  - Receives loop-back from **Append or update row in sheet** to continue with the next batch/item.
- **Edge cases / failures:**
  - If batch size is large, you may hit CompanyEnrich API limits faster.
  - If the loop-back connection is removed/miswired, only the first batch runs.
  - If “Split in Batches” output wiring is wrong (output 0 vs 1), the loop won’t execute as intended.

**Sticky note coverage (contextual):**
- **“Enrichment Loop”**: instructs adding CompanyEnrich API key in HTTP Request.

---

### 2.4 CompanyEnrich lookalike enrichment

**Overview:** Calls CompanyEnrich with the seed domain to obtain similar companies, then explodes the returned list into individual company items.

**Nodes involved:**  
- HTTP Request  
- Split Out  

#### Node: HTTP Request
- **Type / role:** `n8n-nodes-base.httpRequest` — external API call to CompanyEnrich.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.companyenrich.com/companies/similar`
  - **Body (JSON):** `{ "domain": "{{$json.domain}}" }`
  - **Headers (JSON):**
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Content-Type: application/json`
    - `Accept: application/json`
  - `alwaysOutputData: false` (if request errors, node fails rather than passing empty output).
- **Connections:** Input from **Loop Over Items** (batch output), output to **Split Out**.
- **Version specifics:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - Missing/invalid token → 401/403.
  - Invalid/empty domain → 400 or empty results.
  - Rate limiting (429) if many domains processed.
  - Response shape mismatch: downstream expects a field named `items` (see Split Out). If CompanyEnrich returns `data` or `companies` instead, Split Out will fail.

#### Node: Split Out
- **Type / role:** `n8n-nodes-base.splitOut` — converts an array field into separate items.
- **Configuration (interpreted):**
  - **Field to split out:** `items`
  - This means the HTTP response must contain `json.items` as an array.
- **Connections:** Input from **HTTP Request**, output to **Edit Fields1**.
- **Edge cases / failures:**
  - If `items` is missing/not an array, you’ll get no outputs or an error depending on node behavior/version.
  - If the API returns nested arrays/objects, you may need a different field path.

---

### 2.5 Normalization & persistence

**Overview:** Normalizes similar company records into a consistent structure and appends/updates them in Google Sheets using `domain` as the dedupe key.

**Nodes involved:**  
- Edit Fields1  
- Append or update row in sheet  

#### Node: Edit Fields1
- **Type / role:** `n8n-nodes-base.set` — transforms and formats output fields.
- **Configuration (interpreted):**
  - Mode: “raw” and uses a large expression to build an object with many company attributes.
  - Replaces newline characters in all string fields: `v.replace(/\n/g, " ")`.
  - **Important:** It wraps the result in `JSON.stringify(..., null, 2)` and outputs it as the node’s JSON output.
- **Key expressions / variables:**
  - Uses `$json.<field>` for many fields such as `id`, `name`, `domain`, `industry`, `employees`, `revenue`, `logo_url`, `updated_at`, etc.
- **Connections:** Input from **Split Out**, output to **Append or update row in sheet**.
- **Edge cases / failures:**
  - **Potential data-shape issue for Google Sheets mapping:** Because it outputs a **stringified JSON** (a single string) instead of a flat object with fields, Google Sheets “auto map input data” may not find columns like `domain`, `name`, etc. If the intent is to map columns, this node should output actual fields (not a JSON string).
  - Missing fields become `undefined`; Google Sheets may write blanks.
  - If `$json.domain` is missing, dedupe/matching will not work.

#### Node: Append or update row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — persistent storage in a sheet with upsert semantics.
- **Configuration (interpreted):**
  - **Operation:** Append or Update
  - **Document:** Google Spreadsheet “Similar companies” (ID provided)
  - **Sheet tab:** “Sayfa1” (`gid=0`)
  - **Matching column(s):** `domain` (used to detect existing rows and update rather than append)
  - **Mapping mode:** Auto-map input data to the declared schema columns.
- **Credentials:** `googleSheetsOAuth2Api`
- **Connections:**
  - Input from **Edit Fields1**
  - Output loops back to **Loop Over Items** to continue the batch iteration.
- **Version specifics:** `typeVersion: 4.7`
- **Edge cases / failures:**
  - OAuth permission issues (expired token) → auth failure.
  - Sheet/tab name mismatch or deleted tab → “not found”.
  - If `domain` column is missing in the sheet, upsert cannot match and may append duplicates or error.
  - If Edit Fields1 outputs a JSON string (not separate columns), mapping will fail or write everything into one column depending on configuration.

**Sticky note coverage (contextual):**
- **“Update Sheet”**: sheet must contain a `domain` column to avoid duplicates.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow description & prerequisites | — | — | ## Turn Closed-Won Deals in HubSpot into Lookalike Prospects with CompanyEnrich / ### 📌 How This Workflow Works? / 1. **Schedule Trigger:** Executes the workflow start of every week and get your best customers from hubspot crm / 2. **Extract Domain:** Extracts the domain seeds of the companies for CompanyEnrich lookalike companies API. / 3. **Append or Update sheet:** Adds newfound similar companies to your google sheet / ### 🛠️ Before using / ✔ Create an Hubspot private app with company read and write scopes, add your app's credentials to the hubspot node. / ✔ Ensure companies have a valid domain (required for enrichment) / ✔ Create an Company Enrich account and enter your api key to HTTP Request node's header where it says `YOUR_API_KEY`. / ✔ Create a google sheet with a column named 'domain' and connect your google account to the google sheet node... / ### CUSTOMIZATION / 1. **Filters:** Change the Top percent value... / 2. **Schedule:** Change the interval... |
| Schedule Trigger | Schedule Trigger | Weekly workflow entrypoint | — | HubSpot Get Companies | ## Schedule Trigger and Filter / 1. Change the interval at which the workflow runs... / 2. Adjust the **Top_Percent** value... |
| HubSpot Get Companies | HubSpot | Fetch all HubSpot companies | Schedule Trigger | Filter Best | ## Schedule Trigger and Filter / 1. Change the interval... / 2. Adjust the **Top_Percent** value... |
| Filter Best | Function | Select top X% by annual revenue | HubSpot Get Companies | Extract Domain | ## Schedule Trigger and Filter / 1. Change the interval... / 2. Adjust the **Top_Percent** value... |
| Extract Domain | Function | Build `{domain}` seed per company | Filter Best | Loop Over Items |  |
| Loop Over Items | Split In Batches | Iterate through domains (loop controller) | Extract Domain; Append or update row in sheet | HTTP Request (output 1) | ## Enrichment Loop / Enter your CompanyEnrich API key in the **HTTP Request** node where it says `YOUR_API_KEY`. |
| HTTP Request | HTTP Request | Call CompanyEnrich similar-companies API | Loop Over Items | Split Out | ## Enrichment Loop / Enter your CompanyEnrich API key in the **HTTP Request** node where it says `YOUR_API_KEY`. |
| Split Out | Split Out | Split `items[]` array into separate items | HTTP Request | Edit Fields1 |  |
| Edit Fields1 | Set | Normalize/format similar-company fields | Split Out | Append or update row in sheet | ## Update Sheet / ### ⚠️ Avoid Duplicates / Your sheet must contain a column named: domain / The workflow uses this column to detect existing companies |
| Append or update row in sheet | Google Sheets | Upsert similar companies into Google Sheet | Edit Fields1 | Loop Over Items | ## Update Sheet / ### ⚠️ Avoid Duplicates / Your sheet must contain a column named: domain / The workflow uses this column to detect existing companies |
| Sticky Note1 | Sticky Note | Sheet dedupe instructions | — | — | ## Update Sheet / ### ⚠️ Avoid Duplicates / Your sheet must contain a column named: domain / The workflow uses this column to detect existing companies |
| Sticky Note2 | Sticky Note | Schedule + filtering customization | — | — | ## Schedule Trigger and Filter / 1. Change the interval... / 2. Adjust the **Top_Percent** value... |
| Sticky Note3 | Sticky Note | CompanyEnrich API key reminder | — | — | ## Enrichment Loop / Enter your CompanyEnrich API key in the **HTTP Request** node where it says `YOUR_API_KEY`. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   *Turn Closed-Won Deals in HubSpot into Lookalike Prospects with CompanyEnrich*

2. **Add “Schedule Trigger”** (`Schedule Trigger`)
   - Set interval unit to **Weeks** (e.g., every 1 week).
   - This is your entry node.

3. **Add “HubSpot” node** named **HubSpot Get Companies**
   - Resource: **Company**
   - Operation: **Get All**
   - Return All: **true**
   - Authentication: **App Token**
   - Create/select HubSpot credentials:
     - In HubSpot, create a **Private App** and grant at least **CRM companies read** (and any additional scopes you need).
     - Paste token into the n8n HubSpot App Token credential.

4. **Connect**: Schedule Trigger → HubSpot Get Companies

5. **Add a “Function” node** named **Filter Best**
   - Paste code (adjustable):
     - Set `TOP_PERCENT` to desired value (e.g., 5).
     - Sort by `properties.annualrevenue` descending and keep top X%.
   - Connect: HubSpot Get Companies → Filter Best

6. **Add a “Function” node** named **Extract Domain**
   - Configure it to output `{ domain }` using HubSpot properties:
     - Prefer `domain`, fallback to `website`.
   - Connect: Filter Best → Extract Domain

7. **Add “Split In Batches”** node named **Loop Over Items**
   - Set batch size (commonly **1** for safe API pacing).
   - Connect: Extract Domain → Loop Over Items

8. **Add “HTTP Request”** node named **HTTP Request** (CompanyEnrich)
   - Method: **POST**
   - URL: `https://api.companyenrich.com/companies/similar`
   - Body Content Type: **JSON**
   - JSON body:
     - `{"domain": "{{$json.domain}}"}`
   - Headers:
     - `Authorization: Bearer <YOUR_COMPANYENRICH_TOKEN>`
     - `Content-Type: application/json`
     - `Accept: application/json`
   - Connect: Loop Over Items (output for current batch) → HTTP Request

9. **Add “Split Out”** node named **Split Out**
   - Field to split out: `items`
   - Connect: HTTP Request → Split Out  
   - (Confirm CompanyEnrich response indeed contains an `items` array; otherwise adjust the field name.)

10. **Add “Set”** node named **Edit Fields1**
   - Map/normalize fields you want to store (id, name, domain, industry, etc.).
   - Important: to support Google Sheets auto-mapping, ensure the node outputs **actual fields** (e.g., `domain`, `name`, …) rather than a single JSON string.  
   - Connect: Split Out → Edit Fields1

11. **Add “Google Sheets”** node named **Append or update row in sheet**
   - Operation: **Append or Update**
   - Credentials: connect your Google account via OAuth2 in n8n.
   - Select the spreadsheet and the sheet tab.
   - Ensure the sheet has a header column named **`domain`**.
   - Set matching columns to **domain** (dedupe key).
   - Connect: Edit Fields1 → Append or update row in sheet

12. **Close the loop**
   - Connect: Append or update row in sheet → Loop Over Items  
   - This causes the workflow to continue with the next domain until all are processed.

13. **Test**
   - Execute workflow manually once.
   - Verify:
     - HubSpot companies are returned.
     - Filter produces expected count.
     - Domains are non-empty.
     - CompanyEnrich response has `items`.
     - Google Sheet rows are upserted by `domain`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow prerequisites: HubSpot Private App token; CompanyEnrich API key in HTTP headers; Google Sheet with a `domain` column for deduplication; adjust schedule interval and TOP_PERCENT for dataset size. | From workflow sticky notes (“Before using”, “Customization”, “Update Sheet”, “Schedule Trigger and Filter”, “Enrichment Loop”). |