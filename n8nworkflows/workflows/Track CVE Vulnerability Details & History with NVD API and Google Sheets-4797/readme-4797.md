Track CVE Vulnerability Details & History with NVD API and Google Sheets

https://n8nworkflows.xyz/workflows/track-cve-vulnerability-details---history-with-nvd-api-and-google-sheets-4797


# Track CVE Vulnerability Details & History with NVD API and Google Sheets

### 1. Workflow Overview

This workflow automates the retrieval and logging of CVE (Common Vulnerabilities and Exposures) vulnerability details and their change history from the National Vulnerability Database (NVD) API into Google Sheets. It serves security operations teams or vulnerability management processes by providing an easy way to track CVE metadata and historical changes via automated webhooks.

The workflow is logically split into two independent but structurally similar blocks:

- **1.1 CVE Details Retrieval and Logging**:  
  This block listens for incoming webhook requests with a CVE ID, fetches the detailed CVE metadata from the NVD API, processes and flattens the JSON response into a structured format, and appends this data into a Google Sheet titled "NVD Database" in the "CVE Lookup" sheet.

- **1.2 CVE Change History Retrieval and Logging**:  
  This block listens for webhook requests with a CVE ID to fetch the change history of the CVE from the NVD API. It parses the historical change data, flattens it, and appends it into the "CVE History" sheet within the same Google Sheets document.

Both blocks rely on authenticated HTTP requests to the NVD API using API keys managed via HTTP Header Authentication and log data into Google Sheets using OAuth2 credentials.

---

### 2. Block-by-Block Analysis

---

#### 2.1 CVE Details Retrieval and Logging

**Overview:**  
This block accepts webhook requests containing a CVE identifier, fetches detailed CVE information from the NVD API, normalizes the nested JSON into a flat structure, and logs the data into a Google Sheet for easy access and analysis.

**Nodes Involved:**  
- Webhook: Get CVE Details  
- Fetch CVE from NVD API  
- Parse CVE JSON → Flat Format  
- Log CVE Metadata to Sheet  
- Sticky Note1 (NVD API Key info)  
- Sticky Note2 (block description)  
- Sticky Note (general setup instructions)

**Node Details:**

- **Webhook: Get CVE Details**  
  - *Type & Role:* Webhook node; entry point for CVE details retrieval.  
  - *Configuration:* Listens at path `3a3e9d6c-fbbf-482c-a6e9-88cdd79d404a`. Expects a query parameter `cveId`.  
  - *Inputs:* External HTTP requests with CVE ID.  
  - *Outputs:* Passes CVE ID in `query.cveId` to the next node.  
  - *Potential Failures:* Missing or malformed `cveId` query; webhook not reachable.  
  - *Version:* 2.

- **Fetch CVE from NVD API**  
  - *Type & Role:* HTTP Request node; calls NVD API to retrieve CVE metadata.  
  - *Configuration:*  
    - URL: `https://services.nvd.nist.gov/rest/json/cves/2.0`  
    - Query parameter: `cveId` taken dynamically from webhook input (`{{$json.query.cveId}}`).  
    - Authentication: HTTP Header Auth with NVD API key stored in credentials.  
  - *Inputs:* CVE ID from webhook.  
  - *Outputs:* JSON response from NVD API containing CVE data.  
  - *Potential Failures:* API key invalid or expired; network timeout; 404 if CVE not found; rate limiting by NVD.  
  - *Version:* 4.2.

- **Parse CVE JSON → Flat Format**  
  - *Type & Role:* Code node; transforms nested CVE JSON into a flat object suitable for logging.  
  - *Configuration:* Custom JavaScript: iterates over vulnerabilities, extracts key CVE fields such as CVE_ID, description, CVSS metrics, weaknesses, and references.  
  - *Expressions:* Uses optional chaining and array finds to extract English descriptions and CVSS v3.1 data.  
  - *Inputs:* Raw JSON from NVD API HTTP request.  
  - *Outputs:* Structured flat JSON objects for each vulnerability found.  
  - *Potential Failures:* Unexpected JSON structure; undefined fields; empty response.  
  - *Version:* 2.

- **Log CVE Metadata to Sheet**  
  - *Type & Role:* Google Sheets node; appends the parsed CVE metadata to Google Sheets.  
  - *Configuration:*  
    - Operation: Append rows.  
    - Target Sheet: Document ID of the "NVD Database" spreadsheet; Sheet named "CVE Lookup" (gid=0).  
    - Columns mapped automatically based on the parsed data keys (e.g., CVE_ID, Raw_CVE_Data).  
    - Credentials: Google Sheets OAuth2 account configured.  
  - *Inputs:* Flattened CVE metadata JSON from Code node.  
  - *Outputs:* None (terminal).  
  - *Potential Failures:* Authentication failures; quota exceeded; invalid sheet or document ID; network issues.  
  - *Version:* 4.6.

- **Sticky Notes**  
  - Provide contextual information about API key usage, general workflow purpose, and instructions for setup and testing.

---

#### 2.2 CVE Change History Retrieval and Logging

**Overview:**  
This block tracks the change history of CVEs by listening to webhook requests, fetching change history data from the NVD API, parsing and flattening the change events, and logging them into a dedicated Google Sheets tab for historical audit and analysis.

**Nodes Involved:**  
- Webhook: Get CVE Change History  
- Fetch CVE History from NVD API  
- Parse CVE History JSON → Flat Format  
- Log CVE History to Sheet  
- Sticky Note3 (block description)  
- Sticky Note4 (NVD API Key info)  
- Sticky Note (general setup instructions)

**Node Details:**

- **Webhook: Get CVE Change History**  
  - *Type & Role:* Webhook node; entry point for fetching CVE change history.  
  - *Configuration:* Listens at path `587c86f4-4a3a-4765-b486-4bed8a0ccad3`, expects `cveId` query parameter.  
  - *Inputs:* External HTTP requests with CVE ID.  
  - *Outputs:* Passes CVE ID to HTTP request node.  
  - *Potential Failures:* Missing or malformed `cveId`; webhook unreachable.  
  - *Version:* 2.

- **Fetch CVE History from NVD API**  
  - *Type & Role:* HTTP Request node; calls NVD API to retrieve CVE change history.  
  - *Configuration:*  
    - URL: `https://services.nvd.nist.gov/rest/json/cvehistory/2.0`  
    - Query parameter: `cveId` from webhook input.  
    - Authentication: HTTP Header Auth with NVD API key.  
  - *Inputs:* CVE ID from webhook.  
  - *Outputs:* JSON response with change history.  
  - *Potential Failures:* Invalid API key; network issues; rate limits; no history found.  
  - *Version:* 4.2.

- **Parse CVE History JSON → Flat Format**  
  - *Type & Role:* Code node; converts nested change history JSON into flat rows.  
  - *Configuration:* Custom JavaScript: iterates over changes, extracts event details, and if available, individual change details (action, type, old/new values).  
  - *Inputs:* Raw JSON from NVD API.  
  - *Outputs:* Flattened JSON array, each representing a change detail or a change event with empty detail fields.  
  - *Potential Failures:* Unexpected response structure; no changes present.  
  - *Version:* 2.

- **Log CVE History to Sheet**  
  - *Type & Role:* Google Sheets node; appends parsed CVE change history into "CVE History" sheet.  
  - *Configuration:*  
    - Operation: Append rows.  
    - Target Sheet: "NVD Database" spreadsheet, sheet ID 185619006 ("CVE History").  
    - Columns mapped automatically to parsed fields (e.g., CVE_ID, Change_ID, Event, Action, etc.).  
    - Credentials: Google Sheets OAuth2.  
  - *Inputs:* Flattened history data from code node.  
  - *Outputs:* None (terminal).  
  - *Potential Failures:* Authentication errors; sheet quota exceeded; invalid sheet ID; network issues.  
  - *Version:* 4.6.

- **Sticky Notes**  
  - Provide explanations about API key usage and block function for tracking CVE change history.

---

### 3. Summary Table

| Node Name                    | Node Type            | Functional Role                     | Input Node(s)                 | Output Node(s)                | Sticky Note                              |
|------------------------------|----------------------|-----------------------------------|------------------------------|------------------------------|-----------------------------------------|
| Webhook: Get CVE Details      | Webhook              | Entry point for CVE metadata query| –                            | Fetch CVE from NVD API        |                                         |
| Fetch CVE from NVD API        | HTTP Request         | Retrieve CVE metadata from NVD API| Webhook: Get CVE Details      | Parse CVE JSON → Flat Format  | NVD API Key (via HTTP Header Auth) ⬇️  |
| Parse CVE JSON → Flat Format  | Code                 | Flatten CVE JSON data structure   | Fetch CVE from NVD API        | Log CVE Metadata to Sheet     |                                         |
| Log CVE Metadata to Sheet     | Google Sheets        | Append CVE metadata to Google Sheet| Parse CVE JSON → Flat Format | –                            |                                         |
| Webhook: Get CVE Change History| Webhook            | Entry point for CVE change history| –                            | Fetch CVE History from NVD API|                                         |
| Fetch CVE History from NVD API| HTTP Request        | Retrieve CVE change history        | Webhook: Get CVE Change History| Parse CVE History JSON → Flat Format| NVD API Key (via HTTP Header Auth) ⬇️  |
| Parse CVE History JSON → Flat Format | Code           | Flatten CVE change history JSON   | Fetch CVE History from NVD API| Log CVE History to Sheet      |                                         |
| Log CVE History to Sheet      | Google Sheets        | Append CVE change history to sheet| Parse CVE History JSON → Flat Format| –                          |                                         |
| Sticky Note                   | Sticky Note          | General setup and instructions    | –                            | –                            | See detailed setup instructions          |
| Sticky Note1                  | Sticky Note          | NVD API Key info for details block| –                            | –                            | NVD API Key (via HTTP Header Auth) ⬇️  |
| Sticky Note2                  | Sticky Note          | Description for CVE details block | –                            | –                            | 🔍 Lookup CVE vulnerability metadata     |
| Sticky Note3                  | Sticky Note          | Description for CVE history block | –                            | –                            | 🕓 Track change history for CVEs         |
| Sticky Note4                  | Sticky Note          | NVD API Key info for history block| –                            | –                            | NVD API Key (via HTTP Header Auth) ⬇️  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (CVE Details Entry):**  
   - Type: Webhook  
   - Name: `Webhook: Get CVE Details`  
   - Path: `3a3e9d6c-fbbf-482c-a6e9-88cdd79d404a`  
   - No authentication required  
   - Expect query parameter `cveId` (e.g., `?cveId=CVE-XXXX-XXXX`).

2. **Create HTTP Request Node (Fetch CVE Metadata):**  
   - Type: HTTP Request  
   - Name: `Fetch CVE from NVD API`  
   - Method: GET  
   - URL: `https://services.nvd.nist.gov/rest/json/cves/2.0`  
   - Add query parameter `cveId` with expression: `{{$json.query.cveId}}`  
   - Authentication: HTTP Header Auth with NVD API key credential (configure credential with your NVD API key)  
   - Connect output of webhook node to this node.

3. **Create Code Node (Parse CVE JSON):**  
   - Type: Code  
   - Name: `Parse CVE JSON → Flat Format`  
   - Language: JavaScript  
   - Use provided script that iterates over vulnerabilities and extracts relevant fields (refer to the code in the workflow)  
   - Connect output of HTTP Request node to this Code node.

4. **Create Google Sheets Node (Log CVE Metadata):**  
   - Type: Google Sheets  
   - Name: `Log CVE Metadata to Sheet`  
   - Operation: Append rows  
   - Document ID: Your Google Sheets document ID (e.g., `abc1234567890`)  
   - Sheet Name or GID: `gid=0` or "CVE Lookup"  
   - Columns: Map the parsed fields as columns (e.g., CVE_ID, Description, Base_Score, etc.)  
   - Credential: Google Sheets OAuth2 account (set up OAuth2 for your Google account)  
   - Connect output of Code node to this node.

5. **Create Webhook Node (CVE Change History Entry):**  
   - Type: Webhook  
   - Name: `Webhook: Get CVE Change History`  
   - Path: `587c86f4-4a3a-4765-b486-4bed8a0ccad3`  
   - No authentication required  
   - Expect query parameter `cveId`.

6. **Create HTTP Request Node (Fetch CVE History):**  
   - Type: HTTP Request  
   - Name: `Fetch CVE History from NVD API`  
   - Method: GET  
   - URL: `https://services.nvd.nist.gov/rest/json/cvehistory/2.0`  
   - Query parameter `cveId`: `{{$json.query.cveId}}`  
   - Authentication: HTTP Header Auth with same NVD API key credential  
   - Connect output of webhook node to this node.

7. **Create Code Node (Parse CVE History JSON):**  
   - Type: Code  
   - Name: `Parse CVE History JSON → Flat Format`  
   - Language: JavaScript  
   - Use provided script that iterates over change entries and flattens details  
   - Connect output of HTTP Request node to this node.

8. **Create Google Sheets Node (Log CVE History):**  
   - Type: Google Sheets  
   - Name: `Log CVE History to Sheet`  
   - Operation: Append rows  
   - Document ID: Same Google Sheets document ID as above  
   - Sheet Name or GID: `185619006` or "CVE History"  
   - Columns: Map fields like CVE_ID, Change_ID, Event, Action, Old_Value, New_Value, etc.  
   - Credential: Same Google Sheets OAuth2 account  
   - Connect output of Code node to this node.

9. **Sticky Notes (Optional but Recommended):**  
   - Add notes near each block describing their function and credential requirements for clarity and maintenance.

10. **Credentials Setup:**  
    - Create a Google Sheets OAuth2 credential with access to your target spreadsheet.  
    - Create an HTTP Header Auth credential for the NVD API key (store your API key securely).

11. **Testing:**  
    - Trigger the webhooks with HTTP GET requests including a valid CVE ID query parameter, for example:  
      `https://your-n8n-domain/webhook/3a3e9d6c-fbbf-482c-a6e9-88cdd79d404a?cveId=CVE-2023-34362`  
      and  
      `https://your-n8n-domain/webhook/587c86f4-4a3a-4765-b486-4bed8a0ccad3?cveId=CVE-2023-34362`

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| The workflow logs CVE data to Google Sheets titled **NVD Database** with two sheets: **CVE Lookup** (metadata) and **CVE History** (change logs).                                                                                                                                                                              | Google Sheets document configured in nodes.                                                                              |
| Use the official NVD API key for authentication via HTTP Header Auth to avoid rate limiting and access restrictions.                                                                                                                                                                                                           | NVD API documentation: https://nvd.nist.gov/developers/vulnerabilities                                                |
| Sample webhook test URL format: `GET https://your-domain.com/webhook/cve-history?cveId=CVE-2023-34362`                                                                                                                                                                                                                        | Provided in sticky note and setup instructions.                                                                           |
| For best reliability, handle errors such as invalid CVE IDs, API timeouts, and Google Sheets quota limits by adding error workflows or retry mechanisms as needed.                                                                                                                                                           | Suggested best practice for production deployment.                                                                        |
| The workflow leverages n8n version features: Webhook nodes (v2), HTTP Request (v4.2), Google Sheets (v4.6), and Code nodes (v2). Ensure your n8n instance supports these or later versions.                                                                                                                                     | Version requirements noted in node details.                                                                               |
| This workflow is tagged under SecOps and marked as live for active usage.                                                                                                                                                                                                                                                     | Workflow tags metadata.                                                                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automation workflow. All processing complies with applicable content policies and contains no illegal or protected elements. All manipulated data is legal and publicly accessible.