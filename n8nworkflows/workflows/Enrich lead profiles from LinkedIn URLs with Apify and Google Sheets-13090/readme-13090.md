Enrich lead profiles from LinkedIn URLs with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/enrich-lead-profiles-from-linkedin-urls-with-apify-and-google-sheets-13090


# Enrich lead profiles from LinkedIn URLs with Apify and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Enrich lead profiles from LinkedIn URLs with Apify and Google Sheets  
**Internal workflow name:** Get Enriched Lead Profiles from LinkedIn URLs with Apify and Google Sheets

This workflow reads LinkedIn profile URLs from Google Sheets, enriches each lead by scraping the LinkedIn profile via an Apify actor, then writes structured lead data back into the same Google Sheet row. It processes rows sequentially, polls Apify until the scrape run finishes (with retry/timeout logic), and writes either enriched data (“Done”) or an error status.

### 1.1 Input Reception & Row Selection (Google Sheets)
- Manually triggered.
- Fetches only rows that are “not enriched yet” by filtering on an empty/unspecified value in the **First Name** column.

### 1.2 Batch/Sequential Processing
- Uses **Split in Batches** to process **one row at a time**, avoiding rate-limit bursts and making polling simpler.

### 1.3 Apify Run Creation & Polling Loop
- Starts an Apify run for the LinkedIn URL (Apify actor `dev_fusion~linkedin-profile-scraper`).
- Stores run identifiers and metadata.
- Polls Apify every 15 seconds, up to 20 attempts.
- If run succeeds: fetches dataset items.
- If max retries exceeded: writes an error status to the sheet.

### 1.4 Data Extraction & Writeback
- Extracts:
  - Core person + company info (names, headline, about, company details, current role)
  - Additional current roles (if multiple concurrent positions)
  - Most recent previous employer
  - Two latest posts (formatted)
- Merges these computed outputs and updates the same row in Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Manual Start + Fetch Rows to Enrich
**Overview:** Starts the workflow manually and loads unenriched leads from a Google Sheet (rows where “First Name” is empty/unset, acting as the enrichment flag).  
**Nodes involved:**
- When clicking 'Execute Workflow'
- Get Rows - Not Enriched Yet

#### Node: When clicking 'Execute Workflow'
- **Type / role:** Manual Trigger (`manualTrigger`) — entry point.
- **Configuration:** No parameters.
- **Outputs:** Triggers `Get Rows - Not Enriched Yet`.
- **Edge cases:** None (manual-only). Workflow is inactive by default, so must be activated or executed manually.

#### Node: Get Rows - Not Enriched Yet
- **Type / role:** Google Sheets node — reads rows from a spreadsheet.
- **Configuration choices:**
  - **Document:** “Lead Enrichment with LinkedIn Profile URLs” (document ID `1kLOvqW5__U62IinkTCB6R8zVXHdlMayTz4werxnxXtU`)
  - **Sheet:** `Sheet1` (gid=0)
  - **Filter:** Uses `filtersUI` with **lookupColumn = “First Name”**. Interpreted intent: return rows where First Name is empty/not set (i.e., unenriched).
  - Credentials: `Google Sheets account 2` (OAuth2).
- **Outputs:** Rows with `row_number` (used later for updates).
- **Edge cases / failures:**
  - OAuth credential expiration/insufficient permission.
  - Filter behavior depends on node configuration semantics; if filter is misconfigured, it might return all rows or none.
  - Missing expected columns (LinkedIn, First Name, row_number) will break later expressions.

---

### Block 2 — Sequential Row Processing
**Overview:** Processes each row individually to avoid concurrent Apify runs/polling collisions and simplify row-specific updates.  
**Nodes involved:**
- Process One Row at a Time

#### Node: Process One Row at a Time
- **Type / role:** Split In Batches (`splitInBatches`) — controls iteration.
- **Configuration choices:**
  - Default options; processes items sequentially via the node’s loop output.
- **Connections:**
  - Input: `Get Rows - Not Enriched Yet`
  - Output (2nd/main iteration output): `Start Apify Scrape`
  - “Continue” input: fed by both Google Sheets update nodes to advance to next row.
- **Edge cases / failures:**
  - If an error occurs before returning to this node’s “continue” path, the loop stops early.
  - If upstream returns zero rows, nothing executes.

---

### Block 3 — Apify Scraping & Polling (with Retry Limit)
**Overview:** Launches the Apify actor for the LinkedIn URL, stores run metadata, then polls until SUCCEEDED or a retry limit is reached.  
**Nodes involved:**
- Start Apify Scrape
- Store Run Info
- Wait 15 Seconds
- Check Status
- Update Loop Counter
- Check if Ready
- Check Max Retries
- Prepare Error Message
- Update Row in Google Sheet

#### Node: Start Apify Scrape
- **Type / role:** HTTP Request — starts an Apify actor run.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://api.apify.com/v2/acts/dev_fusion~linkedin-profile-scraper/runs?token=YOUR_TOKEN_HERE`
  - **Body (JSON):**
    - `profileUrls: ["{{ $json['LinkedIn'] }}"]` (uses LinkedIn URL from the current sheet row)
  - **executeOnce:** true (node-level setting; prevents multiple executions per workflow run in some contexts).
- **Input / Output:**
  - Input: current item from `Process One Row at a Time` (must include `LinkedIn`)
  - Output: Apify run creation response
- **Edge cases / failures:**
  - Token missing/invalid (`YOUR_TOKEN_HERE` must be replaced).
  - Actor name not found, insufficient Apify permissions, or actor requiring different input schema.
  - LinkedIn URL empty/invalid causing actor failure.
  - Apify rate limits / 429.

#### Node: Store Run Info
- **Type / role:** Code node — normalizes and stores run identifiers + original row metadata for later polling/updates.
- **Key logic:**
  - Reads Apify response: `$input.first().json`
  - Retrieves original row: `$('Process One Row at a Time').item.json`
  - Outputs:
    - `apifyRunId` = `apifyResponse.data.id`
    - `apifyActId` = `apifyResponse.data.actId`
    - `rowNumber` = `originalRow.row_number`
    - `linkedInUrl` = original LinkedIn URL
    - `loopCount` = 0
- **Connections:**
  - Input: `Start Apify Scrape`
  - Output: `Wait 15 Seconds`
- **Edge cases / failures:**
  - If Apify response shape differs (no `data.id`/`data.actId`), expressions throw and polling never starts.
  - If sheet row lacks `row_number`, later updates cannot match the row.

#### Node: Wait 15 Seconds
- **Type / role:** Wait node — delays between status polls.
- **Configuration:** `amount: 15` seconds.
- **Connections:** Output to `Check Status`.
- **Edge cases:**
  - Long workflows can hit execution time limits depending on n8n hosting plan/settings (20 attempts * 15s = ~5 minutes per row, plus overhead).

#### Node: Check Status
- **Type / role:** HTTP Request — fetches Apify run status.
- **Configuration:**
  - **GET URL:** `https://api.apify.com/v2/acts/{{ $json.apifyActId }}/runs/{{ $json.apifyRunId }}?token=YOUR_APIFY_API_KEY`
  - Uses `apifyActId` and `apifyRunId` from previous node output.
- **Connections:** Output to `Update Loop Counter`.
- **Edge cases / failures:**
  - Token missing/invalid (`YOUR_APIFY_API_KEY` must be replaced).
  - Run ID not found/expired.
  - Transient network errors.
  - Note: Using `/acts/{actId}/runs/{runId}` is valid, but Apify commonly uses `/actor-runs/{runId}`; ensure this endpoint works for your Apify API version.

#### Node: Update Loop Counter
- **Type / role:** Code node — increments loop count and extracts `status`.
- **Key logic:**
  - `status` = `statusResponse.data.status`
  - `loopCount = previousData.loopCount + 1`
  - Preserves: `apifyRunId`, `apifyActId`, `rowNumber`, `linkedInUrl`
- **Connections:** Output to `Check if Ready`.
- **Edge cases:**
  - If `statusResponse.data.status` missing, `Check if Ready` may never succeed and retries continue until max.

#### Node: Check if Ready
- **Type / role:** IF node — determines if Apify run succeeded.
- **Condition:** `{{$json.status}} equals "SUCCEEDED"`
- **Outputs:**
  - **True:** to `Fetch LinkedIn Data`
  - **False:** to `Check Max Retries`
- **Edge cases:**
  - If status becomes `FAILED`, this branch will still treat it as “not ready” and continue retrying until max retries, then error.
  - Could be improved by explicitly handling `FAILED`/`ABORTED`.

#### Node: Check Max Retries
- **Type / role:** IF node — enforces retry limit.
- **Condition:** `{{$json.loopCount}} < 20`
- **Outputs:**
  - **True:** loops back to `Wait 15 Seconds`
  - **False:** to `Prepare Error Message`
- **Edge cases:**
  - 20 attempts means up to ~5 minutes waiting per lead; for large sheets, total runtime may be high.

#### Node: Prepare Error Message
- **Type / role:** Code node — prepares an error payload for sheet update.
- **Output fields:**
  - `rowNumber`
  - `linkedInData`: `"Error: Failed after X attempts"`
- **Connection:** to `Update Row in Google Sheet`
- **Edge cases:**
  - `linkedInData` is written to the “Status” column downstream; naming is slightly misleading but functional.

#### Node: Update Row in Google Sheet
- **Type / role:** Google Sheets node — updates the row when scraping times out.
- **Configuration choices:**
  - **Operation:** Update
  - **Match column:** `row_number`
  - **Writes:**
    - `Status = {{$json.linkedInData}}` (error message)
    - `Apify ID = {{ $('Check Status').item.json.data.id }}` (note: this is run id, not dataset id)
    - `row_number = {{$json.rowNumber}}`
  - Same document/sheet/credentials as the read node.
- **Connections:**
  - After update, it returns to `Process One Row at a Time` to continue with next row.
- **Edge cases / failures:**
  - If `Check Status` never ran (e.g., earlier failure), `$('Check Status').item` reference can fail.
  - Column naming mismatch risk: schema includes “LI Key Profile Information” here, but successful path writes “LI Other Profile Information” in the other update node; ensure your sheet columns match what you intend.

---

### Block 4 — Fetch Dataset Items + Data Extraction
**Overview:** Once Apify run is SUCCEEDED, the workflow retrieves the dataset items and transforms them into specific fields and formatted summaries.  
**Nodes involved:**
- Fetch LinkedIn Data
- Get 2 Most Recent Posts
- Get Other Current Jobs
- Set Other Key LinkedIn Data
- Merge

#### Node: Fetch LinkedIn Data
- **Type / role:** HTTP Request — pulls scraped profile JSON items from Apify dataset.
- **Configuration:**
  - **GET URL:** `https://api.apify.com/v2/datasets/{{ $('Check Status').item.json.data.defaultDatasetId }}/items?token=YOUR_APIFY_API_KEY`
  - **alwaysOutputData:** true (ensures node outputs something even on certain error conditions)
- **Connections:** Splits into three parallel processing paths:
  - to `Get 2 Most Recent Posts`
  - to `Get Other Current Jobs`
  - to `Set Other Key LinkedIn Data`
- **Edge cases / failures:**
  - Token missing/invalid.
  - `defaultDatasetId` missing if actor run doesn’t produce a dataset.
  - Dataset may return an array of items; downstream nodes assume a single profile object. If multiple items are returned, n8n will create multiple items and later merge-by-position can misalign.
  - Apify dataset can be empty if scrape failed silently.

#### Node: Get 2 Most Recent Posts
- **Type / role:** Code node — extracts first 2 entries from `updates`.
- **Key logic:**
  - `firstTwoPosts = ($json.updates || []).slice(0, 2)`
  - Formats each post with text + link into a single string `linkedInUpdatesPostsActivity`
- **Output:** `{ linkedInUpdatesPostsActivity: "..." }`
- **Edge cases:**
  - If `updates` is not an array, it falls back to empty and returns “No posts available”.

#### Node: Get Other Current Jobs
- **Type / role:** Code node — builds a formatted list of “additional current roles” beyond the primary current role.
- **Key logic highlights:**
  - Uses `experiences` array safely.
  - Determines “current” role via:
    - `jobStillWorking === true`, else infer by empty `jobEndedOn`
  - Treats first current role as primary; others become `additional_current_role_#` blocks.
  - Normalizes strings, sets missing to `N/A`.
  - Adds a “current as of MM-YYYY” timestamp (timezone fixed to `Asia/Tokyo`).
- **Output fields:**
  - `additionalCurrentJobs` (multi-line text or “No additional current jobs”)
  - `count`, `currentRolesCount`
- **Edge cases:**
  - If Apify actor uses different field names (e.g., `experience` vs `experiences`), this node only checks `experiences` and may miss data.

#### Node: Set Other Key LinkedIn Data
- **Type / role:** Set node — maps raw Apify fields into stable output fields used for Google Sheets update.
- **Notable assignments:**
  - Person: `firstName`, `lastName`, `linkedinUrl`, `linkedInHeadline` (from `headline`), `aboutPersonLead` (from `about`), `leadLocation` (from `addressCountryOnly`), `emailAddress` (from `email`)
  - Dataset id: `apifyLinkedInDatasetID = {{ $('Check Status').item.json.data.defaultDatasetId }}`
  - Company fields: `companyName`, `companyWebsite`, `companySize`, `companyIndustry`
  - Primary current role derived via inline JS in expressions:
    - `latestCompanyName`, `latestJobTitle`, `latestJobDescription`
    - Uses `experiences` or `experience` fallback in some expressions.
  - Previous employer: `oldPreviousCompanyName` computed from experiences order and end date.
  - “latestCompanyIndustry” / “latestCompanySize” are derived from the *same* “primary experience object” if possible, otherwise fallback to top-level.
- **Edge cases / failures:**
  - Extensive inline JS expressions can fail if incoming JSON is unexpectedly shaped (e.g., `experiences` contains non-objects).
  - Field naming inconsistency: one assignment name is `=latestCompanySize` (leading `=`). This is likely a mistake and will create a field literally named `=latestCompanySize`, while later the workflow uses `latestCompanySize` nowhere but uses `companySize` and `latestCompanyIndustry`. Verify intended column mapping.
  - If `Check Status` item is not in scope (execution path differences), dataset ID expression can fail.

#### Node: Merge
- **Type / role:** Merge node — combines the three parallel branches into one item.
- **Configuration:**
  - Mode: `combine`
  - Combine by: `position`
  - Inputs: 3 (jobs, posts, key data)
- **Connections:** Output to `Update Row in Google Sheet (2)`.
- **Edge cases:**
  - If any branch outputs a different number of items, merge-by-position can mismatch (wrong posts paired with wrong profile).
  - If dataset returns multiple profiles, you may end up with multiple merged outputs per row and multiple updates.

---

### Block 5 — Successful Writeback to Google Sheets
**Overview:** Writes enriched data back to the same row, marking status “Done” and filling columns.  
**Nodes involved:**
- Update Row in Google Sheet (2)

#### Node: Update Row in Google Sheet (2)
- **Type / role:** Google Sheets node — updates the enriched row on success.
- **Configuration choices:**
  - **Operation:** Update
  - **Match column:** `row_number`
  - **Writes:**
    - `Status = "Done"`
    - `Add date = {{ new Date().toISOString() }}`
    - `Apify ID = {{ $json.apifyLinkedInDatasetID }}` (dataset id, not run id)
    - `Industry = {{ $json.latestCompanyIndustry }}`
    - `LinkedIn = {{ $json.linkedinUrl }}`
    - `Location = {{ $json.leadLocation }}`
    - `First Name`, `Last Name`
    - `Company Name`, `Company URL`, `Company SIze` (note capitalization/typo), `Job Position`
    - `LI Other Profile Information` is a multi-section formatted text including headline, summary, job description, additional roles, previous employer, and recent posts.
    - `row_number` uses: `{{ $('Get Rows - Not Enriched Yet').item.json.row_number }}` (instead of the current item’s `rowNumber`), which relies on cross-node item linking.
- **Connections:** After update, returns to `Process One Row at a Time` to continue loop.
- **Edge cases / failures:**
  - Column name mismatch: `"Company SIze"` (typo) must match the sheet exactly or the update may fail or write to an unexpected column.
  - Referencing `$('Get Rows - Not Enriched Yet').item.json.row_number` can misalign if multiple items are in play; safer would be using the stored `rowNumber`.
  - Long text field may hit Google Sheets cell limits if posts/descriptions are very long.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute Workflow' | Manual Trigger | Manual entry point | — | Get Rows - Not Enriched Yet | # **Get Enriched Lead Profiles from LinkedIn URLs with Apify and Google Sheets** (purpose, requirements, setup, next steps) |
| Get Rows - Not Enriched Yet | Google Sheets | Read unenriched rows (filter on “First Name”) | When clicking 'Execute Workflow' | Process One Row at a Time | ## 1. Input & Batch Processing (Fetches unenriched rows…) |
| Process One Row at a Time | Split In Batches | Iterate rows sequentially | Get Rows - Not Enriched Yet; Update Row in Google Sheet; Update Row in Google Sheet (2) | Start Apify Scrape | ## 1. Input & Batch Processing (Fetches unenriched rows…) |
| Start Apify Scrape | HTTP Request | Start Apify actor run for LinkedIn URL | Process One Row at a Time | Store Run Info | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Store Run Info | Code | Extract run IDs + row metadata; init loop counter | Start Apify Scrape | Wait 15 Seconds | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Wait 15 Seconds | Wait | Delay between polls | Store Run Info; Check Max Retries (true) | Check Status | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Check Status | HTTP Request | Poll Apify run status | Wait 15 Seconds | Update Loop Counter | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Update Loop Counter | Code | Increment retries; carry status + metadata | Check Status | Check if Ready | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Check if Ready | IF | Branch on `status == SUCCEEDED` | Update Loop Counter | Fetch LinkedIn Data; Check Max Retries | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Check Max Retries | IF | Limit polling to 20 attempts | Check if Ready (false) | Wait 15 Seconds (true); Prepare Error Message (false) | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Prepare Error Message | Code | Build timeout error payload | Check Max Retries (false) | Update Row in Google Sheet | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Update Row in Google Sheet | Google Sheets | Write error status back to row | Prepare Error Message | Process One Row at a Time | ## 2. Apify Scraping & Polling (triggers scrape + polling…) |
| Fetch LinkedIn Data | HTTP Request | Get dataset items from Apify | Check if Ready (true) | Get 2 Most Recent Posts; Get Other Current Jobs; Set Other Key LinkedIn Data | ## 3. Data Extraction & Output (extract + write back…) |
| Get 2 Most Recent Posts | Code | Format first two LinkedIn posts | Fetch LinkedIn Data | Merge (input 2) | ## 3. Data Extraction & Output (extract + write back…) |
| Get Other Current Jobs | Code | Detect + format additional current roles | Fetch LinkedIn Data | Merge (input 1) | ## 3. Data Extraction & Output (extract + write back…) |
| Set Other Key LinkedIn Data | Set | Map core person/company/job fields | Fetch LinkedIn Data | Merge (input 3) | ## 3. Data Extraction & Output (extract + write back…) |
| Merge | Merge | Combine posts + jobs + core fields | Get Other Current Jobs; Get 2 Most Recent Posts; Set Other Key LinkedIn Data | Update Row in Google Sheet (2) | ## 3. Data Extraction & Output (extract + write back…) |
| Update Row in Google Sheet (2) | Google Sheets | Write enriched profile back; mark Done | Merge | Process One Row at a Time | ## 3. Data Extraction & Output (extract + write back…) |
| Sticky Note7 | Sticky Note | Documentation block | — | — | # **Get Enriched Lead Profiles…** (contains requirements, setup steps, tips) |
| Sticky Note | Sticky Note | Sample output reference | — | — | ## Sample Output + [Link to Google Sheets sample file](https://docs.google.com/spreadsheets/d/1kLOvqW5__U62IinkTCB6R8zVXHdlMayTz4werxnxXtU/edit?usp=sharing) + image: https://i.postimg.cc/PxmX6FJw/sample-linkedin-url-lead-profile-enrichment.png |
| Sticky Note1 | Sticky Note | Block label | — | — | ## 1. Input & Batch Processing |
| Sticky Note4 | Sticky Note | Block label | — | — | ## 2. Apify Scraping & Polling |
| Sticky Note5 | Sticky Note | Block label | — | — | ## 3. Data Extraction & Output |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Get Enriched Lead Profiles from LinkedIn URLs with Apify and Google Sheets** (or your preferred name).
   - Keep it **inactive** until credentials and tokens are set.

2. **Add Manual Trigger**
   - Add node: **Manual Trigger**
   - Name: *When clicking 'Execute Workflow'*
   - No configuration needed.

3. **Add Google Sheets (Read) node**
   - Add node: **Google Sheets**
   - Name: *Get Rows - Not Enriched Yet*
   - Credentials: configure **Google Sheets OAuth2** (sign in, grant access).
   - Operation: **Read / Get Many** (exact naming varies by n8n version).
   - Select your **Spreadsheet** and **Sheet**.
   - Configure a filter so only unenriched rows are returned:
     - Use **First Name** as the indicator (rows where “First Name” is empty).
   - Ensure your sheet includes (at minimum):
     - `LinkedIn`, `First Name`, `Last Name`, `Job Position`, `Location`, `Industry`, `Company Name`, `Company URL`, `Company SIze`, `LI Other Profile Information`, `Status`, `Apify ID`, `Add date`, `row_number`

4. **Add Split In Batches**
   - Node: **Split In Batches**
   - Name: *Process One Row at a Time*
   - Batch size: 1 (default behavior for “one at a time” processing).
   - Connect: Manual Trigger → Google Sheets (Read) → Split In Batches

5. **Add HTTP Request to start Apify actor**
   - Node: **HTTP Request**
   - Name: *Start Apify Scrape*
   - Method: **POST**
   - URL: `https://api.apify.com/v2/acts/dev_fusion~linkedin-profile-scraper/runs?token=YOUR_APIFY_API_KEY`
   - Body content type: JSON
   - JSON body:
     - `profileUrls` = array containing the row value from column **LinkedIn**
   - Replace `YOUR_APIFY_API_KEY` with your Apify token (or use n8n credentials + headers if you prefer).
   - Connect: Split In Batches → Start Apify Scrape

6. **Add Code node to store run metadata**
   - Node: **Code**
   - Name: *Store Run Info*
   - Logic: output `apifyRunId`, `apifyActId`, `rowNumber`, `linkedInUrl`, `loopCount=0` using:
     - the Apify run response (`data.id`, `data.actId`)
     - the current sheet row (`row_number`, `LinkedIn`)
   - Connect: Start Apify Scrape → Store Run Info

7. **Add Wait node**
   - Node: **Wait**
   - Name: *Wait 15 Seconds*
   - Amount: **15 seconds**
   - Connect: Store Run Info → Wait

8. **Add HTTP Request to poll run status**
   - Node: **HTTP Request**
   - Name: *Check Status*
   - Method: **GET**
   - URL: `https://api.apify.com/v2/acts/{{apifyActId}}/runs/{{apifyRunId}}?token=YOUR_APIFY_API_KEY`
   - Connect: Wait → Check Status

9. **Add Code node to increment loop counter**
   - Node: **Code**
   - Name: *Update Loop Counter*
   - Logic: carry forward IDs/rowNumber, set `status` from response, increment `loopCount`.
   - Connect: Check Status → Update Loop Counter

10. **Add IF node: run succeeded?**
    - Node: **IF**
    - Name: *Check if Ready*
    - Condition: `status` equals `"SUCCEEDED"`
    - True output → Fetch dataset
    - False output → retry logic
    - Connect: Update Loop Counter → Check if Ready

11. **Add IF node: max retries?**
    - Node: **IF**
    - Name: *Check Max Retries*
    - Condition: `loopCount < 20`
    - True → loop back to Wait
    - False → error handling
    - Connect: Check if Ready (false) → Check Max Retries
    - Connect: Check Max Retries (true) → Wait 15 Seconds

12. **Add Code node: prepare error**
    - Node: **Code**
    - Name: *Prepare Error Message*
    - Output: `rowNumber` and an error message string.
    - Connect: Check Max Retries (false) → Prepare Error Message

13. **Add Google Sheets (Update) node for error**
    - Node: **Google Sheets**
    - Name: *Update Row in Google Sheet*
    - Operation: **Update**
    - Match column: `row_number`
    - Set:
      - `row_number = {{$json.rowNumber}}`
      - `Status = {{$json.linkedInData}}` (the error message)
      - `Apify ID` = run id (optional)
    - Connect: Prepare Error Message → Update Row in Google Sheet
    - Connect: Update Row in Google Sheet → Split In Batches (the “continue” input) to move to next row.

14. **Add HTTP Request to fetch Apify dataset items**
    - Node: **HTTP Request**
    - Name: *Fetch LinkedIn Data*
    - Method: **GET**
    - URL: `https://api.apify.com/v2/datasets/{{defaultDatasetId}}/items?token=YOUR_APIFY_API_KEY`
      - `defaultDatasetId` should come from the prior *Check Status* response (`data.defaultDatasetId`)
    - Connect: Check if Ready (true) → Fetch LinkedIn Data

15. **Add three extraction nodes in parallel**
    - **Code** node: *Get 2 Most Recent Posts* (reads `$json.updates`, formats first 2)
    - **Code** node: *Get Other Current Jobs* (reads `$json.experiences`, formats additional current roles)
    - **Set** node: *Set Other Key LinkedIn Data* (maps person/company/job fields and computes primary role + previous employer)
    - Connect: Fetch LinkedIn Data → each of the three nodes.

16. **Add Merge node**
    - Node: **Merge**
    - Name: *Merge*
    - Mode: **Combine**
    - Combine by: **Position**
    - Number of inputs: **3**
    - Connect outputs:
      - Get Other Current Jobs → Merge input 1
      - Get 2 Most Recent Posts → Merge input 2
      - Set Other Key LinkedIn Data → Merge input 3

17. **Add Google Sheets (Update) node for success**
    - Node: **Google Sheets**
    - Name: *Update Row in Google Sheet (2)*
    - Operation: **Update**
    - Match column: `row_number`
    - Write fields similar to:
      - Status = “Done”
      - Add date = ISO timestamp
      - Apify ID = dataset id
      - First/Last name, Location, Industry, Company details, Job Position
      - LI Other Profile Information = formatted multi-section text
    - Connect: Merge → Update Row in Google Sheet (2)
    - Connect: Update Row in Google Sheet (2) → Split In Batches (continue input)

18. **Replace all Apify placeholders**
    - Replace:
      - `YOUR_TOKEN_HERE`
      - `YOUR_APIFY_API_KEY`
    - Prefer using one consistent token variable or credential approach.

19. **Validate column names**
    - Ensure your sheet has exact matching column headers, especially:
      - `Company SIze` (typo present in the workflow)
      - `LI Other Profile Information` vs `LI Key Profile Information` (the two update nodes reference different names)

20. **Test with 1–3 rows**
    - Add a few LinkedIn URLs, keep **First Name blank**, run manually, verify:
      - Status updates
      - Data fields populated
      - Retry loop behavior

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sample output Google Sheet | https://docs.google.com/spreadsheets/d/1kLOvqW5__U62IinkTCB6R8zVXHdlMayTz4werxnxXtU/edit?usp=sharing |
| Sample output image | https://i.postimg.cc/PxmX6FJw/sample-linkedin-url-lead-profile-enrichment.png |
| Apify actor used: `dev_fusion~linkedin-profile-scraper` | Mentioned in workflow notes and used in the Apify start URL |
| Tip: start with small batches (5–10) to detect rate limiting | Included in the workflow sticky note guidance |

