Enrich Google Sheets with Dun & Bradstreet Data Blocks

https://n8nworkflows.xyz/workflows/enrich-google-sheets-with-dun---bradstreet-data-blocks-8869


# Enrich Google Sheets with Dun & Bradstreet Data Blocks

### 1. Workflow Overview

This workflow automates the enrichment of company data in Google Sheets by integrating with Dun & Bradstreet (D&B) Data Blocks API. It reads company DUNS numbers from a Google Sheet, fetches an OAuth Bearer token from D&B, retrieves payment insight data (specifically the Paydex score) for each company, and then appends or updates the Google Sheet with the enriched data. Rows already marked as `Complete` are skipped to avoid redundant processing.

The workflow logic is grouped into the following blocks:

- **1.1 Input Reception:** Manual trigger initiates the workflow.
- **1.2 Data Retrieval from Google Sheets:** Reads company DUNS data.
- **1.3 Filtering Logic:** Filters out rows already enriched (`Complete` marked).
- **1.4 Authentication:** Obtains D&B API bearer token using Basic Auth.
- **1.5 Data Enrichment:** Calls D&B Data Blocks API to get Paydex scores.
- **1.6 Data Processing:** Extracts Paydex score from API response.
- **1.7 Output Upsert:** Updates or appends the enriched data back to Google Sheets.

Additional sticky notes provide detailed setup instructions and supplementary information about fetching full company reports.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)
- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Allows manual execution to start the workflow  
  - **Configuration:** No parameters required  
  - **Input/Output:** No input; outputs trigger the next node (`Get Companies`)  
  - **Edge Cases:** Manual start means no automatic trigger errors  
  - **Version:** 1  

---

#### 2.2 Data Retrieval from Google Sheets

- **Overview:** Reads rows from a Google Sheet containing company DUNS numbers and enrichment status.
- **Nodes Involved:**  
  - `Get Companies` (Google Sheets)
- **Node Details:**  
  - **Type:** Google Sheets node  
  - **Role:** Fetches all rows from a specific Google Sheets document and tab (`Sheet1`)  
  - **Configuration:**  
    - Document ID: `1h0D5C1oJElBUsz9Dv4AllEUU5eIAR4ae4VTXH1XcwBM` (sheet with raw DUNS data)  
    - Sheet GID: `0` (default tab)  
    - Credential: OAuth2 Google Sheets account  
  - **Key Expressions/Variables:** None (fetches all rows)  
  - **Input:** Trigger from manual start  
  - **Output:** Array of rows with `duns`, `paydex`, `Complete` columns  
  - **Edge Cases:**  
    - Empty or missing sheet  
    - Missing DUNS numbers  
    - Google API rate limiting or auth errors  
  - **Version:** 4.7  

---

#### 2.3 Filtering Logic

- **Overview:** Filters out rows already marked as `Complete` to avoid re-processing.
- **Nodes Involved:**  
  - `Only New Rows` (Filter)
- **Node Details:**  
  - **Type:** Filter node  
  - **Role:** Passes only rows where `Complete` is empty (not yet enriched)  
  - **Configuration:** Condition: `Complete` field is empty string  
  - **Input:** Output from `Get Companies`  
  - **Output:** Filtered rows forwarded to API call  
  - **Edge Cases:**  
    - Rows with missing `Complete` field may pass unnecessarily  
    - Case sensitivity is true; non-empty but non-standard values may cause filtering issues  
  - **Version:** 2.2  

---

#### 2.4 Authentication

- **Overview:** Obtains OAuth 2.0 Bearer token from D&B API using Basic Authentication.
- **Nodes Involved:**  
  - `Get Token1` (HTTP Request)
- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Requests access token from D&B token endpoint  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://plus.dnb.com/v3/token`  
    - Authentication: Basic Auth (D&B username & password credentials)  
    - Body Parameter: `grant_type=client_credentials`  
    - Header: `Content-Type: application/x-www-form-urlencoded`  
  - **Input:** Triggered implicitly by downstream node requiring token  
  - **Output:** JSON with `access_token` field  
  - **Key Expressions:** Bearer token dynamically retrieved for use in next node's Authorization header  
  - **Edge Cases:**  
    - Auth failures due to bad credentials  
    - Network timeouts  
    - Token expiration handled by re-fetch on each execution  
  - **Version:** 1  

---

#### 2.5 Data Enrichment

- **Overview:** Calls D&B Data Blocks API to retrieve payment insight data (Paydex score) for each company DUNS.
- **Nodes Involved:**  
  - `D&B Info` (HTTP Request)
- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Fetches payment insight block for each DUNS number  
  - **Configuration:**  
    - Method: GET  
    - URL Template:  
      ```
      https://plus.dnb.com/v1/data/duns/{{ $json.duns }}?blockIDs=paymentinsight_L4_v1&tradeUp=hq&customerReference=customer%20reference%20text&orderReason=6332
      ```  
    - Headers:  
      - `Accept: application/json`  
      - `Authorization: Bearer {{$node["Get Token1"].json["access_token"]}}` (dynamic token injection)  
    - Authentication: None (token passed via header)  
  - **Input:** Filtered rows from `Only New Rows`  
  - **Output:** JSON response with D&B data blocks per company  
  - **Edge Cases:**  
    - Invalid or expired token  
    - Missing or malformed DUNS number  
    - API rate limits or downtime  
    - Unexpected JSON response structure  
  - **Version:** 1  

---

#### 2.6 Data Processing

- **Overview:** Extracts the Paydex score from the nested API JSON response and prepares data for Google Sheets update.
- **Nodes Involved:**  
  - `Keep Score` (Set node)
- **Node Details:**  
  - **Type:** Set node  
  - **Role:** Creates a new numeric field `Paydex` with extracted score value  
  - **Configuration:**  
    - Field: `Paydex` (number)  
    - Value expression:  
      ```
      {{$json.organization.businessTrading[0].summary[0].paydexScoreHistory[0].paydexScore}}
      ```  
  - **Input:** Output from `D&B Info`  
  - **Output:** JSON enriched with `Paydex` field  
  - **Edge Cases:**  
    - Missing or incomplete nested fields leading to undefined or errors in expression  
    - Cases where `paydexScoreHistory` is empty or missing  
  - **Version:** 3.4  

---

#### 2.7 Output Upsert

- **Overview:** Appends or updates the Google Sheet with the enriched Paydex score and marks the row as `Complete`.
- **Nodes Involved:**  
  - `Append to g-sheets` (Google Sheets)
- **Node Details:**  
  - **Type:** Google Sheets node  
  - **Role:** Appends or updates rows based on matching DUNS number  
  - **Configuration:**  
    - Operation: Append or Update  
    - Matching column: `duns`  
    - Columns mapping:  
      - `duns` mapped from original Google Sheet row (`Get Companies`)  
      - `paydex` from enriched `Paydex` field  
      - `Complete` set to `Yes`  
    - Document ID: `1wlzNuN16KfD72owWceXmTvVulGBplMBv3c1DlsSayHI` (target sheet for enriched data)  
    - Sheet GID: `0`  
    - Credential: OAuth2 Google Sheets account  
  - **Input:** Output from `Keep Score`  
  - **Output:** Updated Google Sheet rows  
  - **Edge Cases:**  
    - Credential or permission errors  
    - Sheet unavailability or schema issues  
    - Data type mismatches  
  - **Version:** 4.7  

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                        | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                   |
|-------------------------------|--------------------|-------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Workflow start trigger               | —                          | Get Companies              |                                                                                                              |
| Get Companies                  | Google Sheets      | Retrieve companies DUNS data         | When clicking ‘Execute workflow’ | Only New Rows              |                                                                                                              |
| Only New Rows                 | Filter             | Filter out already enriched rows     | Get Companies               | D&B Info                   |                                                                                                              |
| Get Token1                    | HTTP Request       | Get D&B Bearer token (Basic Auth)   | —                          | Used dynamically by D&B Info | Sticky Note63: Setup instructions for D&B Auth HTTP Request (Basic Auth for token)                           |
| D&B Info                      | HTTP Request       | Fetch D&B payment insight data      | Only New Rows               | Keep Score                 | Sticky Note55: Overview of workflow purpose and logic for D&B data enrichment                                |
| Keep Score                   | Set                | Extract Paydex score from response   | D&B Info                   | Append to g-sheets         |                                                                                                              |
| Append to g-sheets             | Google Sheets      | Append or update enriched data       | Keep Score                 | —                          | Sticky Note64: Google Sheets OAuth2 credential setup and usage instructions                                  |
| Sticky Note55                 | Sticky Note        | Workflow summary and concept         | —                          | —                          | Overview of enriching DUNS rows with D&B Data Blocks                                                        |
| Sticky Note9                  | Sticky Note        | Setup instructions for entire workflow | —                          | —                          | Detailed setup instructions including Google Sheets and D&B API configuration                               |
| Sticky Note61                 | Sticky Note        | Instructions for fetching company report (PDF/JSON) | —                          | —                          | Details on fetching company reports via D&B API                                                             |
| Sticky Note63                 | Sticky Note        | Setup instructions for D&B Auth HTTP Request | —                          | —                          | Setup info for Basic Auth token retrieval for D&B API                                                       |
| Sticky Note64                 | Sticky Note        | Setup instructions for Google Sheets OAuth2 credential | —                          | —                          | Google Sheets credential connection and sheet setup instructions                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`. No parameters needed.

2. **Add Google Sheets Node to Get Companies:**  
   - Add a **Google Sheets** node named `Get Companies`.  
   - Set **Operation**: default (read rows).  
   - Configure **Document ID** with your source spreadsheet containing DUNS numbers (e.g., `1h0D5C1oJElBUsz9Dv4AllEUU5eIAR4ae4VTXH1XcwBM`).  
   - Set **Sheet Name/GID** to `0` (usually “Sheet1”).  
   - Use Google Sheets OAuth2 credentials configured in n8n.  
   - Connect `When clicking ‘Execute workflow’` output to this node's input.

3. **Add Filter Node to Skip Completed Rows:**  
   - Add a **Filter** node named `Only New Rows`.  
   - Configure condition: `Complete` field is empty string (`=""`).  
   - Connect output of `Get Companies` to `Only New Rows`.

4. **Add HTTP Request Node to Get D&B Token:**  
   - Add an **HTTP Request** node named `Get Token1`.  
   - Set **Method**: POST.  
   - URL: `https://plus.dnb.com/v3/token`.  
   - Authentication: Basic Auth with D&B username & password credentials.  
   - Under **Body Parameters**, add `grant_type` with value `client_credentials`.  
   - Add header: `Content-Type: application/x-www-form-urlencoded`.  
   - No input connection needed; this node will be called dynamically.

5. **Add HTTP Request Node to Call D&B Data Blocks:**  
   - Add an **HTTP Request** node named `D&B Info`.  
   - Set **Method**: GET.  
   - URL Template:  
     ```
     https://plus.dnb.com/v1/data/duns/{{ $json.duns }}?blockIDs=paymentinsight_L4_v1&tradeUp=hq&customerReference=customer%20reference%20text&orderReason=6332
     ```  
   - Authentication: None (token passed in header).  
   - Add Headers:  
     - `Accept: application/json`  
     - `Authorization: Bearer {{$node["Get Token1"].json["access_token"]}}` (dynamic expression referencing token node)  
   - Connect `Only New Rows` output to this node's input.

6. **Add Set Node to Extract Paydex Score:**  
   - Add a **Set** node named `Keep Score`.  
   - Add a new field called `Paydex` (type: Number).  
   - Set value expression:  
     ```
     {{$json.organization.businessTrading[0].summary[0].paydexScoreHistory[0].paydexScore}}
     ```  
   - Connect `D&B Info` output to this node.

7. **Add Google Sheets Node to Append or Update Data:**  
   - Add a **Google Sheets** node named `Append to g-sheets`.  
   - Set **Operation** to `Append or Update`.  
   - Set **Matching Column** to `duns`.  
   - Map columns:  
     - `duns` → `={{ $('Get Companies').item.json.duns }}`  
     - `paydex` → `={{ $json.Paydex }}`  
     - `Complete` → `Yes` (static value)  
   - Configure **Document ID** to your target spreadsheet for enriched data (e.g., `1wlzNuN16KfD72owWceXmTvVulGBplMBv3c1DlsSayHI`).  
   - Set **Sheet Name/GID** to `0`.  
   - Use Google Sheets OAuth2 credentials (can be different from source sheet).  
   - Connect `Keep Score` output to this node.

8. **Connect the Workflow:**  
   - Connect `When clicking ‘Execute workflow’` → `Get Companies` → `Only New Rows` → `D&B Info` → `Keep Score` → `Append to g-sheets`.

9. **Credential Setup:**  
   - Configure Google Sheets OAuth2 credentials with access to the relevant spreadsheets.  
   - Configure Basic Auth credentials for D&B API with your username and password.

10. **Test the Workflow:**  
    - Execute manually and verify rows with empty `Complete` column are enriched with `paydex` scores and marked `Yes`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates enrichment of company DUNS in Google Sheets using Dun & Bradstreet Data Blocks API with dynamic bearer token fetching and append-or-update logic.                                                                                                                                                                                                          | General workflow overview                                                                        |
| Setup instructions include obtaining Google Sheets OAuth2 credentials and D&B Basic Auth credentials. Token retrieval uses `client_credentials` grant type.                                                                                                                                                                                                                   | Sticky Note9, Sticky Note63                                                                     |
| D&B Data Blocks API endpoint for payment insight block: `/v1/data/duns/{{duns}}?blockIDs=paymentinsight_L4_v1&tradeUp=hq&customerReference=customer%20reference%20text&orderReason=6332`                                                                                                                                                                                         | Sticky Note55                                                                                   |
| Paydex score is extracted from a nested JSON path: `organization.businessTrading[0].summary[0].paydexScoreHistory[0].paydexScore`. Edge cases include missing or empty arrays that may cause expression failures.                                                                                                                                                                  | Node `Keep Score` details                                                                       |
| To fetch full company reports (PDF/JSON), a separate HTTP Request node can be used with appropriate parameters; outputs can be routed to Google Drive or email for compliance.                                                                                                                                                                                                | Sticky Note61                                                                                   |
| Security reminder: Do not hardcode tokens; always fetch dynamically to avoid token expiration issues.                                                                                                                                                                                                                                                                          | Sticky Note9                                                                                   |
| For contact or customization help, reach out to Robert Breen via email or LinkedIn.                                                                                                                                                                                                                                                                                            | Email: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/      |
| Sample spreadsheet URLs and IDs are placeholders; replace them with your own sheets when reproducing the workflow.                                                                                                                                                                                                                                                            | Sticky Note64                                                                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.