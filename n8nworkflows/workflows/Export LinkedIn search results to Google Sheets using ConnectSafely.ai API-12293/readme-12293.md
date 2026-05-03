Export LinkedIn search results to Google Sheets using ConnectSafely.ai API

https://n8nworkflows.xyz/workflows/export-linkedin-search-results-to-google-sheets-using-connectsafely-ai-api-12293


# Export LinkedIn search results to Google Sheets using ConnectSafely.ai API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Export LinkedIn search results to Google Sheets using ConnectSafely.ai API

**Purpose:**  
This workflow searches LinkedIn people profiles via the **ConnectSafely.ai API** (without cookies), normalizes the response into a consistent “export” schema (similar to common scraping/export tools), and then exports the results to:
- **Google Sheets** (append rows)
- **A JSON file** (for download/storage downstream)

**Target use cases:**
- Prospecting lists (e.g., “CEO SaaS in United States”)
- Building lead databases in Google Sheets
- Creating machine-readable exports (JSON) for enrichment pipelines

### Logical blocks
**1.1 Trigger & Parameter Input**  
Manual trigger starts the workflow and sets search criteria.

**1.2 LinkedIn Search via ConnectSafely.ai API**  
HTTP POST to `/linkedin/search/people` using Bearer token auth.

**1.3 Normalization / Formatting**  
Code node converts the API response into a row-per-person structure and extracts a best-effort company name.

**1.4 Export (Sheets + JSON)**  
Appends rows to a target Google Sheet and also generates a timestamped JSON file.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Parameter Input

**Overview:**  
Starts the workflow manually and defines the LinkedIn search parameters (keywords, location, title, limit) that will be used in the API request.

**Nodes involved:**
- Manual Trigger
- Set Search Parameters

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual entry point for testing and ad-hoc runs.
- **Configuration choices:** No parameters; it simply emits one empty item.
- **Inputs / outputs:**
  - **Output →** Set Search Parameters
- **Edge cases / failures:** None (only depends on n8n being reachable).

#### Node: Set Search Parameters
- **Type / role:** `n8n-nodes-base.set` — creates/overwrites fields used for the API request payload.
- **Configuration choices (interpreted):**
  - Sets these fields on the item:
    - `keywords` = `"CEO SaaS"`
    - `location` = `"United States"`
    - `title` = `"Head of Growth"`
    - `limit` = `"100"` (**stored as string**, later injected as a number in JSON body)
- **Key variables / expressions:** None; static values.
- **Inputs / outputs:**
  - **Input ←** Manual Trigger
  - **Output →** Search LinkedIn People
- **Edge cases / failures:**
  - `limit` is defined as a **string**. The HTTP node uses it unquoted in JSON (`"limit": {{ $json.limit }}`), which works if the content is numeric (e.g., `"100"`). If set to a non-numeric string (e.g., `"ten"`), the request body becomes invalid JSON and the HTTP request will fail.

---

### 2.2 LinkedIn Search via ConnectSafely.ai API

**Overview:**  
Calls ConnectSafely.ai to perform a LinkedIn people search using the parameters set in the previous node.

**Nodes involved:**
- Search LinkedIn People

#### Node: Search LinkedIn People
- **Type / role:** `n8n-nodes-base.httpRequest` — performs the external API call.
- **Configuration choices (interpreted):**
  - **Method:** POST  
  - **URL:** `https://api.connectsafely.ai/linkedin/search/people`
  - **Body type:** JSON body enabled
  - **Response format:** JSON
  - **Authentication:** Generic credential type → **HTTP Bearer Auth**
- **Key expressions / variables used (body):**
  ```js
  {
    "keywords": "{{ $json.keywords }}",
    "location": "{{ $json.location }}",
    "title": "{{ $json.title }}",
    "limit": {{ $json.limit }}
  }
  ```
- **Credentials:**
  - Uses **HTTP Bearer Auth** credential named: `Anandi Devi  ConnectSafely AI`
  - Note: The node JSON also contains an `httpHeaderAuth` credential reference, but the node is configured for Bearer auth. In practice, only the selected auth method is used.
- **Inputs / outputs:**
  - **Input ←** Set Search Parameters
  - **Output →** Format Results
- **Edge cases / potential failures:**
  - **401/403** if API key missing/invalid or plan restrictions.
  - **429** rate limiting (bursting searches).
  - **5xx** ConnectSafely.ai downtime.
  - **Invalid JSON body** if `limit` becomes non-numeric (see previous block).
  - **Schema drift**: if response fields change (e.g., `people` renamed), downstream formatting will yield empty results.

---

### 2.3 Normalization / Formatting

**Overview:**  
Converts the API response into one output item per person, with stable column names suitable for Google Sheets export. Attempts to extract a company name from a headline string.

**Nodes involved:**
- Format Results

#### Node: Format Results
- **Type / role:** `n8n-nodes-base.code` — JavaScript transformation of input items to normalized output items.
- **Configuration choices (interpreted):**
  - Reads **all input items** (`$input.all()`).
  - Handles the API returning either:
    - an array with a single object (`[ { success, people, ... } ]`), or
    - a direct object (`{ success, people, ... }`)
  - Only processes entries where `data.success` is truthy.
  - Iterates `data.people` and emits one item per person.
  - Extracts `company` using a regex over `headline` or `currentPosition`.
  - Adds `extractedAt` as ISO timestamp.
  - If no people are produced, returns a single item: `{ error: 'No results found', timestamp: ... }`
- **Key logic / variables:**
  - Company extraction regex:
    - Looks for patterns like “at Company”, “@ Company”, or “- Company”
    - Stops before `|` or `•` separators
  - Output fields produced per person:
    - `profileUrl`, `profileId`, `profileUrn`
    - `fullName`, `firstName`, `lastName`
    - `headline`, `currentPosition`, `company`
    - `location`, `connectionDegree`
    - `isPremium`, `isOpenToWork`
    - `profilePicture`
    - `extractedAt`
- **Inputs / outputs:**
  - **Input ←** Search LinkedIn People
  - **Output →** Export to Google Sheets
  - **Output →** Export to JSON
- **Edge cases / potential failures:**
  - If API returns **no `success` flag** or a different shape, all results may be skipped.
  - If no people found, it emits an **error object** that will still be exported to Google Sheets/JSON unless filtered (currently not filtered).
  - Company extraction is heuristic; headlines not matching “at/@/-” patterns will produce empty `company`.
  - Code node runtime errors possible if response is not JSON (but HTTP node is set to JSON).

---

### 2.4 Export (Sheets + JSON)

**Overview:**  
Exports the normalized people rows to Google Sheets (append) and simultaneously creates a JSON file containing the output.

**Nodes involved:**
- Export to Google Sheets
- Export to JSON

#### Node: Export to Google Sheets
- **Type / role:** `n8n-nodes-base.googleSheets` — appends rows to a spreadsheet.
- **Configuration choices (interpreted):**
  - **Operation:** Append
  - **Document:** “LinkedIn Search Export Data” (Spreadsheet ID `1rSor29CdaLpmtLCFaJhFaMwBYA-Ih1PeuG3iw1iHy5E`)
  - **Sheet/tab:** `Sheet1` (`gid=0`)
  - **Mapping mode:** Auto-map input data to defined schema columns
  - **Columns schema includes** (not exhaustive): `profileUrl`, `fullName`, `firstName`, `lastName`, `headline`, `currentTitle`, `company`, `location`, `connectionDegree`, `isPremium`, `isOpenToWork`, `profilePicture`, `extractedAt`, etc.
- **Important mapping note:**
  - The formatter outputs `currentPosition`, but the sheet schema includes `currentTitle` (and not `currentPosition`).
  - With auto-mapping, this means **current position/title may not populate** unless the sheet column name matches the incoming field (or you change the code to output `currentTitle`).
- **Credentials:**
  - Google Sheets OAuth2 credential: `Google Sheets account`
- **Inputs / outputs:**
  - **Input ←** Format Results
  - **Output:** none (terminal)
- **Edge cases / potential failures:**
  - OAuth token expired / revoked → auth error.
  - Spreadsheet permissions insufficient (needs edit access).
  - Column mismatch: fields not present in sheet schema won’t be written.
  - If the “No results found” error item is produced, it can append a row with mostly empty columns unless you add filtering.

#### Node: Export to JSON
- **Type / role:** `n8n-nodes-base.convertToFile` — converts items to a downloadable/storable JSON file (binary output).
- **Configuration choices (interpreted):**
  - **Operation:** Convert to JSON
  - **Filename expression:** `linkedin-export-{{ $now.format('yyyy-MM-dd-HHmmss') }}.json`
- **Inputs / outputs:**
  - **Input ←** Format Results
  - **Output:** binary JSON file (terminal unless connected to storage/email/etc.)
- **Edge cases / potential failures:**
  - If upstream returns the single `{error: ...}` item, the JSON file will contain that instead of people rows.
  - Large result sets can create large JSON; may be impacted by n8n memory limits depending on hosting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Entry point (manual run) | — | Set Search Parameters |  |
| Set Search Parameters | Set | Define search criteria (keywords/location/title/limit) | Manual Trigger | Search LinkedIn People | **Search** — Configure parameters and call ConnectSafely.ai API |
| Search LinkedIn People | HTTP Request | Call ConnectSafely.ai LinkedIn people search endpoint | Set Search Parameters | Format Results | **Search** — Configure parameters and call ConnectSafely.ai API |
| Format Results | Code | Normalize API response into row-per-person schema | Search LinkedIn People | Export to Google Sheets; Export to JSON | **Export** — Format results and save to Sheets/JSON |
| Export to Google Sheets | Google Sheets | Append normalized rows into a spreadsheet | Format Results | — | **Export** — Format results and save to Sheets/JSON |
| Export to JSON | Convert to File | Generate timestamped JSON file from output items | Format Results | — | **Export** — Format results and save to Sheets/JSON |
| Instructions | Sticky Note | Inline operational notes / setup info | — | — | @[youtube](N4kZ3YiLh2Q) / Setup steps / Features (see Section 5) |
| 1️⃣ Search | Sticky Note | Visual grouping label | — | — | **Search** — Configure parameters and call ConnectSafely.ai API |
| 2️⃣ Export | Sticky Note | Visual grouping label | — | — | **Export** — Format results and save to Sheets/JSON |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add node: “Manual Trigger”**
   - Node type: **Manual Trigger**
   - No configuration needed.

3. **Add node: “Set Search Parameters”**
   - Node type: **Set**
   - Add fields:
     - `keywords` (String) — e.g., `CEO SaaS`
     - `location` (String) — e.g., `United States`
     - `title` (String) — e.g., `Head of Growth`
     - `limit` (String or Number)
       - Recommended: store as **Number** to avoid invalid JSON (e.g., `100`).
   - Connect: **Manual Trigger → Set Search Parameters**

4. **Create credentials for ConnectSafely.ai**
   - Go to **Credentials → New**
   - Choose **HTTP Bearer Auth**
   - Paste your ConnectSafely.ai API key as the Bearer token
   - Save (name it as you prefer)

5. **Add node: “Search LinkedIn People”**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.connectsafely.ai/linkedin/search/people`
   - Authentication: **Generic Credential Type → HTTP Bearer Auth**
     - Select the credential created in step 4
   - Body:
     - Enable **Send Body**
     - Choose **JSON**
     - Use this structure (with expressions):
       - `keywords`: `{{$json.keywords}}`
       - `location`: `{{$json.location}}`
       - `title`: `{{$json.title}}`
       - `limit`: `{{$json.limit}}`
   - Response:
     - Set response format to **JSON**
   - Connect: **Set Search Parameters → Search LinkedIn People**

6. **Add node: “Format Results”**
   - Node type: **Code**
   - Paste the transformation logic (same behavior as provided):
     - Accepts array/object wrapper
     - Checks `success`
     - Iterates `people`
     - Outputs one item per person with standardized fields
     - Emits `{error: ...}` item if none found
   - Connect: **Search LinkedIn People → Format Results**

7. **Create credentials for Google Sheets**
   - Go to **Credentials → New**
   - Choose **Google Sheets OAuth2**
   - Authenticate with a Google account that has edit access to the target spreadsheet.

8. **Add node: “Export to Google Sheets”**
   - Node type: **Google Sheets**
   - Operation: **Append**
   - Select your **Spreadsheet** (Document ID)
   - Select the **Sheet/Tab**
   - Column mapping:
     - Use **Auto-map input data**
     - Ensure your sheet headers match the formatter output fields.
     - Important: either
       - change sheet column `currentTitle` → `currentPosition`, or
       - change the code output to emit `currentTitle` instead of `currentPosition`.
   - Connect: **Format Results → Export to Google Sheets**

9. **Add node: “Export to JSON”**
   - Node type: **Convert to File**
   - Operation: **To JSON**
   - Filename: `linkedin-export-{{$now.format('yyyy-MM-dd-HHmmss')}}.json`
   - Connect: **Format Results → Export to JSON**

10. (Optional but recommended) **Add filtering to avoid exporting error rows**
   - Insert an **IF** node between “Format Results” and exports:
     - Condition: `{{$json.error}}` *does not exist*
   - Route “true” to exports; route “false” to a notification/logging step.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| YouTube video reference: `@[youtube](N4kZ3YiLh2Q)` | From the “Instructions” sticky note |
| Get API key: connectSafely dashboard → Settings → API Keys | https://connectsafely.ai/dashboard |
| Requires credentials: HTTP Bearer Auth (ConnectSafely.ai) + Google Sheets OAuth2 | Setup requirement |
| Update Sheet ID in export node to your spreadsheet | Google Sheets node configuration |
| Features listed: Search by keywords/location/title; platform-compliant (no cookies); up to 100+ results; dual export Sheets + JSON | From the “Instructions” sticky note |