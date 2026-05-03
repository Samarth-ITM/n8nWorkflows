Find similar B2B companies to your best customers with Google Sheets and CompanyEnrich

https://n8nworkflows.xyz/workflows/find-similar-b2b-companies-to-your-best-customers-with-google-sheets-and-companyenrich-12300


# Find similar B2B companies to your best customers with Google Sheets and CompanyEnrich

## 1. Workflow Overview

**Purpose:**  
This workflow reads a list of B2B company domains (your “best customers” or target accounts) from Google Sheets, calls the **CompanyEnrich** “similar companies” endpoint for each domain, and writes the returned lookalike companies (name, domain, similarity score) into a results tab in the same spreadsheet.

**Target use cases:**
- Expanding an account list from known high-quality customers/leads
- Building prospecting lists based on firmographic similarity signals returned by CompanyEnrich
- Creating a repeatable enrichment pipeline from Sheets → API → Sheets

### 1.1 Input Reception (Trigger + Source Data)
- Manual start
- Reads a sheet column named **`Domain`** that contains the source company websites/domains

### 1.2 Similar-Company Enrichment (API Call + Merge)
- For each source domain, sends an HTTP POST to CompanyEnrich `/companies/similar`
- Merges the API response back with the original row data so the source domain stays available downstream

### 1.3 Result Normalization + Persistence (Split + Append)
- Splits the API’s list of similar companies into one item per similar company
- Appends a row per similar company into a destination results sheet

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Trigger + Read Sheet)

**Overview:**  
Starts the workflow manually and loads the list of source domains from Google Sheets for processing.

**Nodes involved:**
- Manual Trigger
- Read Source List

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — entry point for manual runs.
- **Configuration choices:** No parameters.
- **Inputs / outputs:**  
  - Output → **Read Source List**
- **Edge cases / failures:** None (only used for manual execution).
- **Version notes:** TypeVersion `1`.

#### Node: Read Source List
- **Type / role:** `n8n-nodes-base.googleSheets` — reads input rows from a Google Sheet tab (“Source List”).
- **Configuration choices (interpreted):**
  - Uses a specific Spreadsheet: **“Find Company Lookalikes”** (documentId `148uMDDvJLyCKwgClg8tdaMDwxYPXPxsv6EWCyH11R4s`)
  - Reads from sheet/tab: **“Source List”** (`gid=0`)
  - Operation is not explicitly shown in JSON snippet, but this node is used as a **read** source of rows.
- **Key fields/variables expected:**
  - Each row should contain a column named **`Domain`** (used later as `{{$json.Domain}}`).
- **Inputs / outputs:**
  - Input ← **Manual Trigger**
  - Output 1 (main index 0) → **Fetch Similar Companies**
  - Output 2 (main index 1) → **Merge Source & API** (Merge input 2)
- **Credentials:**
  - Google Sheets OAuth2: **“Google Sheets account 2”**
- **Edge cases / failure types:**
  - Missing/invalid OAuth credentials, revoked access
  - Spreadsheet/tab not found
  - Missing `Domain` column or empty cells → downstream API call may fail or return no results
  - Large sheet sizes may hit API/sheets performance limits
- **Version notes:** TypeVersion `4.7` (fields/operations vary across versions).

---

### Block 2 — Enrich & Process (API Call + Merge)

**Overview:**  
Calls CompanyEnrich to fetch similar companies for each source domain, then merges the API response with the original source row so both datasets are available for final output.

**Nodes involved:**
- Fetch Similar Companies
- Merge Source & API

#### Node: Fetch Similar Companies
- **Type / role:** `n8n-nodes-base.httpRequest` — POST request to CompanyEnrich similar-companies endpoint.
- **Configuration choices (interpreted):**
  - **Method:** POST  
  - **URL:** `https://api.companyenrich.com/companies/similar`
  - **Headers:** `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder)
  - **Body:** `domain` = `{{$json.Domain}}`
  - **Error handling:** `onError: continueRegularOutput`  
    This means failed requests won’t stop the workflow; the node will continue emitting output (often including error data depending on n8n’s HTTP Request node behavior/version).
- **Key expressions/variables used:**
  - `={{ $json.Domain }}` to populate the request body.
- **Inputs / outputs:**
  - Input ← **Read Source List** (main index 0)
  - Output → **Merge Source & API** (Merge input 1)
- **Edge cases / failure types:**
  - Invalid/expired API token → 401/403
  - Rate limits / throttling → 429
  - Timeout/network/DNS errors
  - Bad input domain formatting (e.g., includes protocol, spaces) → API may reject or reduce match quality
  - Because errors are continued, downstream nodes may receive unexpected shapes (e.g., missing `items`, missing `metadata`)
- **Version notes:** TypeVersion `4.2`.

#### Node: Merge Source & API
- **Type / role:** `n8n-nodes-base.merge` — combines the original sheet row with its corresponding API response.
- **Configuration choices (interpreted):**
  - **Mode:** Combine
  - **Combine by:** Position (row 1 with response 1, etc.)
  - This assumes the HTTP Request returns items in the same order and count as the input items.
- **Inputs / outputs:**
  - Input 1 ← **Fetch Similar Companies**
  - Input 2 ← **Read Source List** (main index 1)
  - Output → **Split Out Items**
- **Edge cases / failure types:**
  - If the HTTP node returns fewer/more items than the sheet read (e.g., due to filtered errors), position-based merge can misalign source domains to the wrong API responses.
  - If HTTP errors produce items with different structure, downstream splitting/mapping can fail.
- **Version notes:** TypeVersion `3`.

---

### Block 3 — Save Results (Split + Append to Sheet)

**Overview:**  
Transforms the API list of similar companies into one row per company and appends those rows to a “Lookalike Results” sheet.

**Nodes involved:**
- Split Out Items
- Write Results

#### Node: Split Out Items
- **Type / role:** `n8n-nodes-base.splitOut` — expands an array field into multiple items.
- **Configuration choices (interpreted):**
  - **Field to split out:** `items`
  - **Include:** all other fields  
    This keeps fields from the merged record (e.g., `Domain`, `metadata`) while iterating each element of `items`.
- **Inputs / outputs:**
  - Input ← **Merge Source & API**
  - Output → **Write Results**
- **Edge cases / failure types:**
  - If `items` is missing, null, or not an array → node may output zero items or error (version/behavior dependent).
  - If the API returns an empty list for some domains, nothing will be written for those domains.
- **Version notes:** TypeVersion `1`.

#### Node: Write Results
- **Type / role:** `n8n-nodes-base.googleSheets` — appends rows into the results sheet.
- **Configuration choices (interpreted):**
  - **Operation:** Append
  - **Spreadsheet:** same as input (`148uMDDvJLyCKwgClg8tdaMDwxYPXPxsv6EWCyH11R4s`)
  - **Sheet/tab:** “Lookalike Results” (`gid=1540563700`)
  - **Column mapping (define below):**
    - **Added Date:** `{{$now}}`
    - **Source Domain:** `{{$json.Domain}}`
    - **Similar Domain:** `{{$json.items.domain}}`
    - **Similarity Score:** `{{$json.metadata.scores[$json.items.id]}}`
    - **Similar Company Name:** `{{$json.items.name}}`
  - Type conversion disabled (keeps values as-is).
- **Inputs / outputs:**
  - Input ← **Split Out Items**
  - Output: end of workflow
- **Credentials:**
  - Google Sheets OAuth2: **“Google Sheets account 2”**
- **Edge cases / failure types:**
  - If `metadata.scores` or `items.id` is missing, the similarity score expression may evaluate to `null`/empty (or error if strict evaluation changes).
  - If the destination sheet doesn’t have the exact columns, append may fail or write blanks depending on mapping.
  - Google Sheets API quota limits for large volumes.
- **Version notes:** TypeVersion `4.7`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual entry point | — | Read Source List | ## 1. Get Input Data<br>This section triggers the workflow and reads the list of target domains from your Google Sheet. |
| Read Source List | Google Sheets | Read source domains from “Source List” tab | Manual Trigger | Fetch Similar Companies; Merge Source & API | ## 1. Get Input Data<br>This section triggers the workflow and reads the list of target domains from your Google Sheet. |
| Fetch Similar Companies | HTTP Request | Call CompanyEnrich similar-companies API | Read Source List | Merge Source & API | ## 2. Enrich & Process<br>Queries the API to find lookalikes, merges the results with the original data, and splits the list so each similar company gets its own row. |
| Merge Source & API | Merge | Combine original row + API response by position | Fetch Similar Companies; Read Source List | Split Out Items | ## 2. Enrich & Process<br>Queries the API to find lookalikes, merges the results with the original data, and splits the list so each similar company gets its own row. |
| Split Out Items | Split Out | Convert `items[]` array into individual rows | Merge Source & API | Write Results | ## 2. Enrich & Process<br>Queries the API to find lookalikes, merges the results with the original data, and splits the list so each similar company gets its own row. |
| Write Results | Google Sheets | Append lookalike rows into results tab | Split Out Items | — | ## 3. Save Results<br>Writes the newly discovered companies, their domains, and similarity scores into your destination sheet. |
| Sticky Note | Sticky Note | Documentation / usage notes | — | — | ## How it works<br>This workflow expands your target list by finding companies similar to your existing leads.<br>1. **Reads Data:** It grabs a list of domains from your Google Sheet.<br>2. **Enriches:** It queries the CompanyEnrich API to find "lookalike" companies for each domain.<br>3. **Formats:** It processes the JSON response to separate individual company results into rows.<br>4. **Saves:** Finally, it appends the new company details (Name, Domain, Similarity Score) to your results sheet.<br><br>## Setup steps<br>1. **Google Sheets:** Create a spreadsheet. In the first tab, create a column named `Domain` and list your target websites. Create a second empty tab for the results.<br>2. **Configure Sheets Nodes:** Open the "Read Source List" and "Write Results" nodes. Select your specific spreadsheet and the respective tabs.<br>3. **API Key:** Open the "Fetch Similar Companies" node. In the Headers section, replace the placeholder in `Authorization` with your actual CompanyEnrich API Key (Bearer Token).<br><br>## Beginner Guide<br>You can watch the quick guide to get started @[youtube](guD5yxlKm0o) |
| Sticky Note1 | Sticky Note | Documentation for block 1 | — | — | ## 1. Get Input Data<br>This section triggers the workflow and reads the list of target domains from your Google Sheet. |
| Sticky Note2 | Sticky Note | Documentation for block 2 | — | — | ## 2. Enrich & Process<br>Queries the API to find lookalikes, merges the results with the original data, and splits the list so each similar company gets its own row. |
| Sticky Note3 | Sticky Note | Documentation for block 3 | — | — | ## 3. Save Results<br>Writes the newly discovered companies, their domains, and similarity scores into your destination sheet. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Google Spreadsheet**
   1. Create a spreadsheet (e.g., “Find Company Lookalikes”).
   2. Tab 1: name it **Source List**. Add a header column **`Domain`** and fill it with company domains (one per row).
   3. Tab 2: name it **Lookalike Results**. Add headers:
      - `Added Date`
      - `Source Domain`
      - `Similar Domain`
      - `Similarity Score`
      - `Similar Company Name`

2. **Create node: Manual Trigger**
   - Add **Manual Trigger** node.
   - No configuration needed.

3. **Create node: Read Source List (Google Sheets)**
   - Add **Google Sheets** node.
   - **Credentials:** connect a Google Sheets OAuth2 credential (authorize access).
   - **Document:** select your spreadsheet.
   - **Sheet:** select **Source List**.
   - Configure it to **read/get all rows** (use the default “Read” operation for your node version).
   - Ensure output items contain `Domain`.

4. **Create node: Fetch Similar Companies (HTTP Request)**
   - Add **HTTP Request** node.
   - Set:
     - **Method:** POST  
     - **URL:** `https://api.companyenrich.com/companies/similar`
     - **Send Headers:** enabled  
       - Header `Authorization` = `Bearer <YOUR_COMPANYENRICH_TOKEN>`
     - **Send Body:** enabled (JSON/body parameters)
       - Body field `domain` = expression `{{$json.Domain}}`
   - Set **Error handling** to continue (equivalent to “Continue on Fail”) so one bad domain doesn’t stop the run.

5. **Create node: Merge Source & API (Merge)**
   - Add **Merge** node.
   - Set **Mode** to **Combine**.
   - Set **Combine By** to **Position**.
   - This node must receive:
     - Input 1: API response
     - Input 2: original source rows

6. **Create node: Split Out Items (Split Out)**
   - Add **Split Out** node.
   - Set **Field to split out:** `items`
   - Set it to **include all other fields** (so `Domain` and `metadata` remain available).

7. **Create node: Write Results (Google Sheets)**
   - Add **Google Sheets** node.
   - **Credentials:** same Google OAuth2 credential as reading (or another with access).
   - **Document:** same spreadsheet.
   - **Sheet:** **Lookalike Results**
   - **Operation:** Append
   - Map columns using expressions:
     - Added Date → `{{$now}}`
     - Source Domain → `{{$json.Domain}}`
     - Similar Domain → `{{$json.items.domain}}`
     - Similarity Score → `{{$json.metadata.scores[$json.items.id]}}`
     - Similar Company Name → `{{$json.items.name}}`

8. **Connect nodes in this order**
   1. Manual Trigger → Read Source List
   2. Read Source List → Fetch Similar Companies
   3. Read Source List (second output) → Merge Source & API (input 2)
   4. Fetch Similar Companies → Merge Source & API (input 1)
   5. Merge Source & API → Split Out Items
   6. Split Out Items → Write Results

9. **Run and validate**
   - Execute manually.
   - Confirm new rows appear in **Lookalike Results** with expected fields.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow requires a `Domain` column in the source sheet and a separate results tab. | Setup guidance from sticky note |
| Replace `Authorization: Bearer YOUR_TOKEN_HERE` with your real CompanyEnrich Bearer token. | CompanyEnrich API authentication |
| “Beginner Guide” link from the workflow note: `@[youtube](guD5yxlKm0o)` (video id). | Likely YouTube URL: https://www.youtube.com/watch?v=guD5yxlKm0o |
| Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Compliance / provenance statement |