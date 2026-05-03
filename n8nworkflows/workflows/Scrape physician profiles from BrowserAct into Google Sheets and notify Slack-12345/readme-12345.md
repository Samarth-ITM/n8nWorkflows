Scrape physician profiles from BrowserAct into Google Sheets and notify Slack

https://n8nworkflows.xyz/workflows/scrape-physician-profiles-from-browseract-into-google-sheets-and-notify-slack-12345


# Scrape physician profiles from BrowserAct into Google Sheets and notify Slack

## 1. Workflow Overview

**Purpose:** This workflow scrapes physician profiles (Name, Address, Specialty) from the Healow website using a BrowserAct automation, stores/updates the results in Google Sheets, and notifies a Slack channel when finished.

**Primary use cases:**
- Enriching/maintaining a physician directory for a specific city/state
- Periodic lead capture and deduplication (via “append or update” on Google Sheets)
- Operational visibility via Slack notification

### 1.1 Input Reception (On-demand)
Manual start; sets the run context for a chosen location.

### 1.2 Target & Scrape (BrowserAct)
Passes City/State into a BrowserAct workflow (“Healow” automation) to extract data.

### 1.3 Parse, Archive, Notify
Parses the BrowserAct output string into individual n8n items, writes them into Google Sheets (append or update), and posts a completion message to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (On-demand)
**Overview:** Starts the workflow manually and provides an entry point for testing and ad-hoc runs.  
**Nodes involved:** `On-Demand Execution`

#### Node: On-Demand Execution
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — workflow entry point.
- **Configuration:** No parameters.
- **Inputs/outputs:** No inputs; outputs to **Define Location**.
- **Edge cases/failures:** None (only requires user execution).
- **Version notes:** TypeVersion 1.

---

### Block 2 — Target & Scrape (BrowserAct)
**Overview:** Defines the geographic search parameters and runs a BrowserAct workflow that scrapes Healow and returns the scraped data payload.  
**Nodes involved:** `Define Location`, `Run BrowserAct Workflow`

#### Node: Define Location
- **Type / role:** Set node (`n8n-nodes-base.set`) — injects fixed parameters (City/State).
- **Configuration choices:**
  - Creates two string fields:
    - `Location` = `"Brooklyn"`
    - `State` = `"NY"`
- **Key variables/expressions:** None (static values).
- **Inputs/outputs:** Input from **On-Demand Execution**; output to **Run BrowserAct Workflow**.
- **Edge cases/failures:**
  - Incorrect location/state values may lead to empty scrape results downstream.
- **Version notes:** TypeVersion 3.4.

#### Node: Run BrowserAct Workflow
- **Type / role:** BrowserAct node (`n8n-nodes-browseract.browserAct`) — executes a BrowserAct-hosted workflow automation.
- **Configuration choices (interpreted):**
  - **Mode:** `WORKFLOW`
  - **BrowserAct workflowId:** `71087742787674119`
  - **Timeout:** 7200 seconds (2 hours), suitable for longer scraping runs.
  - **Workflow inputs passed:**
    - `input-Location` = `{{ $json.Location }}`
    - `input-State` = `{{ $json.State }}`
  - Incognito mode: disabled (`open_incognito_mode: false`)
- **Credentials:** BrowserAct API credential (“BrowserAct account”) required.
- **Inputs/outputs:**
  - Input from **Define Location**
  - Output to **Splitting Items**
- **Edge cases/failures:**
  - BrowserAct auth/permission errors (invalid API key, workflow not accessible).
  - WorkflowId mismatch (deleted/renamed workflow in BrowserAct).
  - Timeout if Healow is slow, blocked, or BrowserAct run hangs.
  - Output schema mismatch: downstream expects a specific output path (`$input.first().json.output.string`).
- **Sub-workflow reference:** Invokes BrowserAct workflow `71087742787674119` (template referenced in notes as “Physician Profile Enricher” / Healow template).
- **Version notes:** TypeVersion 1.

---

### Block 3 — Parse, Archive, Notify
**Overview:** Converts the BrowserAct returned JSON string into separate items, upserts them into Google Sheets (match on Name), then sends a Slack completion notification.  
**Nodes involved:** `Splitting Items`, `Update Leads`, `Alert Team`

#### Node: Splitting Items
- **Type / role:** Code node (`n8n-nodes-base.code`) — parses JSON string and emits one item per physician record.
- **Configuration choices (interpreted):**
  - Reads: `const jsonString = $input.first().json.output.string;`
  - Validates presence of `jsonString`; throws a hard error if missing.
  - `JSON.parse(jsonString)`; throws a hard error on malformed JSON.
  - Ensures parsed result is an array; throws error if not.
  - Outputs: `parsedData.map(item => ({ json: item }))`
- **Key variables/expressions:**
  - `$input.first().json.output.string` is critical and must match BrowserAct output structure.
- **Inputs/outputs:**
  - Input from **Run BrowserAct Workflow**
  - Output to **Update Leads**
- **Edge cases/failures:**
  - If BrowserAct returns an object instead of an array, workflow fails (“Parsed data is not an array…”).
  - If BrowserAct returns empty string or null, workflow fails immediately.
  - If BrowserAct returns already-structured JSON (not a string), parsing will fail.
- **Version notes:** TypeVersion 2.

#### Node: Update Leads
- **Type / role:** Google Sheets node (`n8n-nodes-base.googleSheets`) — writes rows using upsert semantics.
- **Configuration choices (interpreted):**
  - **Document:** Google Sheet with ID `164YfKm0Jiwy2KNUyX18ugWxbKM9xfXhA2BHg8VPg8vU`
  - **Sheet/Tab:** “Physician Profile” (gid `243879573`)
  - **Operation:** `appendOrUpdate`
  - **Matching column:** `Name` (used to decide update vs append)
  - **Column mapping (expressions):**
    - `Name` = `{{ $json.item[0].Name }}`
    - `Address` = `{{ $json.item[0].Address }}`
    - `Specialty` = `{{ $json.item[0].Specialty }}`
- **Credentials:** Google Sheets OAuth2 credential required.
- **Inputs/outputs:**
  - Input from **Splitting Items**
  - Output to **Alert Team**
- **Important note (likely mapping issue):**
  - `Splitting Items` outputs each physician record as `item.json = { Name, Address, Specialty, ... }`.
  - But this node maps from `{{ $json.item[0].Name }}` which implies the record is nested under `item[0]`.
  - Unless BrowserAct’s parsed objects actually have an `item: [{...}]` structure, this will write blanks.
  - Typical mapping after the Code node should be `{{ $json.Name }}`, `{{ $json.Address }}`, `{{ $json.Specialty }}`.
- **Edge cases/failures:**
  - OAuth scope/permission errors, sheet not found, tab name mismatch.
  - Headers must exist exactly (Name, Specialty, Address) or mapping may fail/append unexpected columns.
  - Upsert collisions if “Name” is not unique (two physicians with same name).
- **Version notes:** TypeVersion 4.7.

#### Node: Alert Team
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — sends a message to a channel.
- **Configuration choices (interpreted):**
  - Posts text: `Physician Profile Enricher Workflow Finished succesfully`
  - Target: channel `all-browseract-workflow-test` (channel ID `C09KLV9DJSX`)
  - `executeOnce: true` set on node (prevents repeated sends in some multi-item contexts, depending on n8n behavior/version).
- **Credentials:** Slack API credential required.
- **Inputs/outputs:** Input from **Update Leads**; no outgoing connection.
- **Edge cases/failures:**
  - Slack auth errors, missing channel permissions, channel archived.
  - If `executeOnce` is not supported/behaves differently across versions, you may get one message per item unless controlled.
- **Version notes:** TypeVersion 2.4.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On-Demand Execution | Manual Trigger | Entry point (manual run) | — | Define Location | ### 🏥 Step 1: Target & Scrape  \nSets the search parameters (City and State) and executes the **Healow** BrowserAct automation to extract physician data from the target website. |
| Define Location | Set | Defines City/State parameters | On-Demand Execution | Run BrowserAct Workflow | ### 🏥 Step 1: Target & Scrape  \nSets the search parameters (City and State) and executes the **Healow** BrowserAct automation to extract physician data from the target website. |
| Run BrowserAct Workflow | BrowserAct | Executes BrowserAct scraping workflow | Define Location | Splitting Items | ### 🏥 Step 1: Target & Scrape  \nSets the search parameters (City and State) and executes the **Healow** BrowserAct automation to extract physician data from the target website. |
| Splitting Items | Code | Parse JSON string and split into items | Run BrowserAct Workflow | Update Leads | ### 💾 Step 2: Process & Archive  \nParses the raw scraped string into structured JSON objects, updates the Google Sheet database with new records, and sends a Slack notification upon success. |
| Update Leads | Google Sheets | Append or update rows (dedupe by Name) | Splitting Items | Alert Team | ### 💾 Step 2: Process & Archive  \nParses the raw scraped string into structured JSON objects, updates the Google Sheet database with new records, and sends a Slack notification upon success.  \n### Google Sheet Headers  \nTo use this workflow, create a Google Sheet with the following headers:  \n* Name  \n* Specialty  \n* Address |
| Alert Team | Slack | Notify Slack channel on completion | Update Leads | — | ### 💾 Step 2: Process & Archive  \nParses the raw scraped string into structured JSON objects, updates the Google Sheet database with new records, and sends a Slack notification upon success. |
| Documentation | Sticky Note | Documentation / setup notes | — | — | ## ⚡ Workflow Overview & Setup  \n\n**Summary:** Automate the extraction of physician profiles (Name, Address, Specialty) from **Healow** based on location, archiving data to Google Sheets and notifying via Slack.\n\n### Requirements\n* **Credentials:** BrowserAct, Google Sheets, Slack.\n* **Mandatory:** BrowserAct API (Template: **Physician Profile Enricher**)\n\n### How to Use\n1. **Credentials:** Configure your API keys for BrowserAct, Google Sheets, and Slack.\n2. **BrowserAct Template:** Ensure you have the **Healow** template saved in your BrowserAct account.\n3. **Google Sheets:** Create a sheet named \"Physician Profile\" with headers: `Name`, `Address`, `Specialty`.\n4. **Configuration:** Update the **Define Location** node with your target City and State.\n\n### Need Help?\n[How to Find Your BrowseAct API Key & Workflow ID](https://docs.browseract.com)\n[How to Connect n8n to Browseract](https://docs.browseract.com)\n[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | Sticky Note | Visual annotation | — | — | ### 🏥 Step 1: Target & Scrape  \n\nSets the search parameters (City and State) and executes the **Healow** BrowserAct automation to extract physician data from the target website. |
| Step 2 Explanation | Sticky Note | Visual annotation | — | — | ### 💾 Step 2: Process & Archive  \n\nParses the raw scraped string into structured JSON objects, updates the Google Sheet database with new records, and sends a Slack notification upon success. |
| Sticky Note | Sticky Note | Visual annotation (Sheet headers) | — | — | ### Google Sheet Headers  \nTo use this workflow, create a Google Sheet with the following headers:\n* Name\n* Specialty\n* Address |
| Sticky Note1 | Sticky Note | Video link | — | — | @[youtube](DZ_Jq_b2-Ww) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“Physician Profile Enricher”** (or your preferred name).

2. **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `On-Demand Execution`

3. **Add Set node for location**
   - Node: **Set**
   - Name: `Define Location`
   - Add fields:
     - `Location` (String) = `Brooklyn`
     - `State` (String) = `NY`
   - Connect: `On-Demand Execution` → `Define Location`

4. **Add BrowserAct node to run the scraping automation**
   - Node: **BrowserAct**
   - Name: `Run BrowserAct Workflow`
   - Configure:
     - Type/Mode: **WORKFLOW**
     - Workflow ID: `71087742787674119`
     - Timeout: `7200`
     - Inputs mapping:
       - `input-Location` = expression `{{ $json.Location }}`
       - `input-State` = expression `{{ $json.State }}`
   - Credentials:
     - Create/select **BrowserAct API** credentials in n8n (API key from BrowserAct).
   - Connect: `Define Location` → `Run BrowserAct Workflow`
   - BrowserAct side requirement:
     - In your BrowserAct account, ensure the Healow scraping workflow/template exists and accepts `Location` and `State` inputs (names aligned with `input-Location` and `input-State`).

5. **Add Code node to parse/split BrowserAct output**
   - Node: **Code**
   - Name: `Splitting Items`
   - Paste logic (conceptually):
     - Read `jsonString` from `{{$input.first().json.output.string}}`
     - `JSON.parse()`
     - Validate it is an array
     - Return `[{json: record1}, {json: record2}, ...]`
   - Connect: `Run BrowserAct Workflow` → `Splitting Items`

6. **Add Google Sheets node (append or update)**
   - Node: **Google Sheets**
   - Name: `Update Leads`
   - Operation: **Append or Update**
   - Select the spreadsheet and sheet/tab:
     - Spreadsheet: create or select a doc (example name: “Physician Profile Enricher”)
     - Tab name: **“Physician Profile”**
   - Ensure the sheet has headers exactly:
     - `Name`, `Address`, `Specialty`
   - Matching column(s): `Name`
   - Map columns:
     - If your Code node outputs `{Name, Address, Specialty}` at top level, set:
       - `Name` = `{{ $json.Name }}`
       - `Address` = `{{ $json.Address }}`
       - `Specialty` = `{{ $json.Specialty }}`
     - (Only use `{{ $json.item[0].Name }}` style mapping if your items truly contain `item: [ ... ]`.)
   - Credentials:
     - Create/select **Google Sheets OAuth2** credentials.
   - Connect: `Splitting Items` → `Update Leads`

7. **Add Slack node for notification**
   - Node: **Slack**
   - Name: `Alert Team`
   - Action: **Post message** (send text to channel)
   - Channel: choose your target channel
   - Text: `Physician Profile Enricher Workflow Finished succesfully`
   - Credentials:
     - Create/select **Slack API** credentials (OAuth).
   - Connect: `Update Leads` → `Alert Team`
   - Optional: configure to send only once per run (if your Slack node/n8n version supports an equivalent of `executeOnce`).

8. **(Optional) Add sticky notes**
   - Add sticky notes with:
     - Setup requirements (BrowserAct/Sheets/Slack)
     - Header requirements
     - Video link: `@[youtube](DZ_Jq_b2-Ww)`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automate extraction of physician profiles (Name, Address, Specialty) from Healow; archive to Google Sheets; notify Slack. | Workflow intent (from sticky note) |
| Credentials required: BrowserAct, Google Sheets, Slack. | Setup requirement |
| BrowserAct template/workflow must exist (Healow template saved in BrowserAct). | BrowserAct prerequisite |
| Google Sheet tab should be named “Physician Profile” with headers: Name, Address, Specialty. | Data storage prerequisite |
| How to Find Your BrowseAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to Browseract | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | @[youtube](DZ_Jq_b2-Ww) |