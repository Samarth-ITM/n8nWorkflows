Track US Fintech & Healthtech Funding Rounds: Crunchbase to Google Sheets

https://n8nworkflows.xyz/workflows/track-us-fintech---healthtech-funding-rounds--crunchbase-to-google-sheets-4796


# Track US Fintech & Healthtech Funding Rounds: Crunchbase to Google Sheets

### 1. Workflow Overview

This workflow automates the daily retrieval and logging of new funding rounds in the US fintech and healthtech sectors from Crunchbase, storing the results in a Google Sheets document for easy tracking and analysis.

**Target Use Cases:**  
- Investors or analysts monitoring US fintech and healthtech funding activities.  
- Business intelligence teams maintaining up-to-date funding data.  
- Automating data ingestion from Crunchbase API to spreadsheet for reporting or further processing.

**Logical Blocks:**  
- **1.1 Scheduled Trigger**: Initiates the workflow daily at a specified time.  
- **1.2 Data Retrieval**: Fetches the latest funding rounds from Crunchbase API with specific filters applied.  
- **1.3 Data Processing**: Extracts and formats relevant funding round details from the API response.  
- **1.4 Data Storage**: Appends the processed data to a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow once daily at 8:00 AM to initiate the data retrieval process.

- **Nodes Involved:**  
  - Daily Check for New Funding Rounds

- **Node Details:**  
  - **Name:** Daily Check for New Funding Rounds  
  - **Type:** Schedule Trigger  
  - **Configuration:** Set to trigger every day at hour 8 (8:00 AM) local time.  
  - **Expressions/Variables:** None.  
  - **Input Connections:** None (entry point).  
  - **Output Connections:** Connected to “Fetch Crunchbase Funding Rounds”.  
  - **Version Requirements:** Compatible with n8n version supporting Schedule Trigger v1.2.  
  - **Edge Cases:**  
    - Workflow will not trigger if n8n instance is offline at scheduled time.  
    - Timezone assumptions depend on n8n server settings; ensure correct timezone configured.  
  - **Sub-workflow Reference:** None.

#### 1.2 Data Retrieval

- **Overview:**  
  Requests the most recent 25 US fintech and healthtech funding rounds from Crunchbase API, sorted by creation date descending.

- **Nodes Involved:**  
  - Fetch Crunchbase Funding Rounds

- **Node Details:**  
  - **Name:** Fetch Crunchbase Funding Rounds  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - HTTP Method: GET (default for request with query parameters)  
    - URL: `https://api.crunchbase.com/api/v4/entities/funding_rounds`  
    - Query Parameters:  
      - locations = United States  
      - industry = Fintech,Healthtech  
      - sort_order = created_at DESC  
      - page = 1  
      - items_per_page = 25  
    - Headers: Includes `X-cb-user-key` with API key placeholder `YOUR_API_KEY`.  
  - **Expressions/Variables:** None used; static query and header values except API key.  
  - **Input Connections:** Receives trigger from “Daily Check for New Funding Rounds”.  
  - **Output Connections:** Passes API response to “Extract & Format Funding Data”.  
  - **Version Requirements:** HTTP Request node version 4.2 or higher recommended for handling query parameters properly.  
  - **Edge Cases:**  
    - API authentication failure if API key is missing or invalid (401 Unauthorized).  
    - API rate limiting or downtime could cause request failures or timeouts.  
    - Network connectivity issues.  
  - **Sub-workflow Reference:** None.

#### 1.3 Data Processing

- **Overview:**  
  Parses the Crunchbase API response JSON, extracts and formats key funding round details into a simplified, consistent structure suitable for spreadsheet insertion.

- **Nodes Involved:**  
  - Extract & Format Funding Data

- **Node Details:**  
  - **Name:** Extract & Format Funding Data  
  - **Type:** Code (JavaScript)  
  - **Configuration:** Custom JS code that iterates over the API response's `entities` array, extracting:  
    - Company name (from funded organization identifier)  
    - Industry groups (joined string)  
    - Funding round type  
    - Announcement date  
    - Amount raised in USD  
    - Investors (joined string)  
    - Crunchbase URL constructed from base URL + permalink  
  - **Expressions/Variables:** Uses `items[0].json.data.entities` input; maps fields carefully with fallback defaults (e.g., “N/A” or 0).  
  - **Input Connections:** Receives data from “Fetch Crunchbase Funding Rounds”.  
  - **Output Connections:** Output is an array of simplified JSON objects passed to “Save to Google Sheets”.  
  - **Version Requirements:** Code node version 2 or higher recommended for modern JS syntax support.  
  - **Edge Cases:**  
    - Missing or undefined properties handled gracefully with defaults.  
    - Empty or malformed API response could cause code errors; error handling not explicitly implemented.  
  - **Sub-workflow Reference:** None.

#### 1.4 Data Storage

- **Overview:**  
  Appends the processed funding data to a specified Google Sheets document and sheet, mapping fields automatically based on column headers.

- **Nodes Involved:**  
  - Save to Google Sheets

- **Node Details:**  
  - **Name:** Save to Google Sheets  
  - **Type:** Google Sheets  
  - **Configuration:**  
    - Operation: Append data rows  
    - Document ID: Placeholder `YOUR_GOOGLE_SHEET_ID` (must be replaced with actual spreadsheet ID)  
    - Sheet Name: Uses sheet with `gid=0` (default first sheet)  
    - Columns Mapping: Auto-mapped from input JSON to columns:  
      - company_name → Company Name  
      - industry → Industry  
      - funding_round_type → Funding Round Type  
      - announced_date → Announced Date  
      - money_raised_usd → Money Raised (USD)  
      - investors → Investors  
      - crunchbase_url → Crunchbase URL  
  - **Expressions/Variables:** Sheet and document IDs use expression list mode for dynamic selection but statically configured here.  
  - **Input Connections:** Receives formatted data from “Extract & Format Funding Data”.  
  - **Output Connections:** None (terminal node).  
  - **Version Requirements:** Google Sheets node v3 or higher recommended.  
  - **Edge Cases:**  
    - Authentication errors if Google credentials are invalid or expired.  
    - Spreadsheet ID or sheet name errors if incorrect or inaccessible.  
    - API quota limits or connectivity failures.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role            | Input Node(s)                 | Output Node(s)               | Sticky Note                                         |
|------------------------------|--------------------|----------------------------|------------------------------|-----------------------------|----------------------------------------------------|
| Daily Check for New Funding Rounds | Schedule Trigger   | Initiates daily workflow   | —                            | Fetch Crunchbase Funding Rounds |                                                    |
| Fetch Crunchbase Funding Rounds    | HTTP Request       | Retrieves funding data     | Daily Check for New Funding Rounds | Extract & Format Funding Data |                                                    |
| Extract & Format Funding Data      | Code               | Parses and formats API data| Fetch Crunchbase Funding Rounds | Save to Google Sheets         |                                                    |
| Save to Google Sheets              | Google Sheets      | Appends data to spreadsheet| Extract & Format Funding Data | —                           |                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named `Crunchbase funding rounds`.

2. **Add a Schedule Trigger node:**  
   - Name: `Daily Check for New Funding Rounds`  
   - Set to trigger daily at 08:00 AM.  
   - Save.

3. **Add an HTTP Request node:**  
   - Name: `Fetch Crunchbase Funding Rounds`  
   - Set URL to `https://api.crunchbase.com/api/v4/entities/funding_rounds`.  
   - Method: GET (default).  
   - Add Query Parameters:  
     - `locations` = `United States`  
     - `industry` = `Fintech,Healthtech`  
     - `sort_order` = `created_at DESC`  
     - `page` = `1`  
     - `items_per_page` = `25`  
   - Add Header Parameter:  
     - `X-cb-user-key` = Your Crunchbase API key (replace `YOUR_API_KEY`).  
   - Connect `Daily Check for New Funding Rounds` output to this node input.

4. **Add a Code node:**  
   - Name: `Extract & Format Funding Data`  
   - Paste the following JavaScript code into the code editor:

```javascript
const baseUrl = "https://www.crunchbase.com/funding-round/";

const output = [];

for (const item of items[0].json.data.entities) {
  const props = item.properties;
  const id = item.identifier;

  const companyName = props.funded_organization_identifier?.value || "N/A";
  const industry = props.industry_group_identifiers?.map(ind => ind.value).join(", ") || "N/A";
  const fundingType = props.funding_type || "N/A";
  const date = props.announced_on || "N/A";
  const amount = props.money_raised_usd || 0;
  const investors = props.investor_identifiers?.map(inv => inv.value).join(", ") || "N/A";
  const url = baseUrl + id.permalink;

  output.push({
    json: {
      company_name: companyName,
      industry: industry,
      funding_round_type: fundingType,
      announced_date: date,
      money_raised_usd: amount,
      investors: investors,
      crunchbase_url: url
    }
  });
}

return output;
```
   - Connect `Fetch Crunchbase Funding Rounds` output to this node input.

5. **Add a Google Sheets node:**  
   - Name: `Save to Google Sheets`  
   - Operation: Append  
   - Document ID: Enter your Google Sheets document ID in place of `YOUR_GOOGLE_SHEET_ID`.  
   - Sheet Name: Use default sheet with `gid=0` or specify your target sheet name.  
   - Columns Mapping: Enable automatic column mapping and map the following fields:  
     - `company_name` → Company Name  
     - `industry` → Industry  
     - `funding_round_type` → Funding Round Type  
     - `announced_date` → Announced Date  
     - `money_raised_usd` → Money Raised (USD)  
     - `investors` → Investors  
     - `crunchbase_url` → Crunchbase URL  
   - Connect `Extract & Format Funding Data` output to this node input.

6. **Configure credentials:**  
   - For the HTTP Request node, no special credentials needed beyond the Crunchbase API key in headers.  
   - For Google Sheets node, create or select existing Google OAuth2 credentials with access to the target spreadsheet.

7. **Activate** the workflow to enable daily execution.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Crunchbase API key is required and must be valid with sufficient quota.                                 | https://data.crunchbase.com/docs/getting-started                                                       |
| Google Sheets document ID must be accessible by configured Google OAuth2 credentials.                   | https://developers.google.com/sheets/api/guides/concepts                                              |
| Timezone for schedule trigger depends on n8n server configuration; verify to ensure correct execution time.| n8n documentation on Schedule Trigger node                                                             |
| The Crunchbase API URL and parameters are based on v4 API endpoints and schema; verify compatibility regularly.| https://data.crunchbase.com/docs/api                                                                 |
| For large volumes, consider pagination beyond page=1 or incremental updates to avoid duplicates.       | Workflow currently fetches only first 25 entries sorted by creation date                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly available.