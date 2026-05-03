Export & Organize Technology Stack Data From BuiltWith to Google Sheets

https://n8nworkflows.xyz/workflows/export---organize-technology-stack-data-from-builtwith-to-google-sheets-4785


# Export & Organize Technology Stack Data From BuiltWith to Google Sheets

### 1. Workflow Overview

This workflow automates the extraction and organization of technology stack data from the BuiltWith API for a list of domains stored in a Google Sheets document. It is primarily designed for marketing, sales intelligence, market analysis, or competitive research use cases where understanding the technological makeup of websites is valuable.

The workflow consists of three main logical blocks:

- **1.1 Input Collection**: Starts the workflow manually and reads domain names from a Google Sheets spreadsheet.
- **1.2 Tech Stack Lookup via BuiltWith API**: For each domain, fetches detailed technology stack data using BuiltWith’s API and extracts key information.
- **1.3 Output Data to Google Sheets**: Updates the original Google Sheets with extracted technology stack details aligned to each domain row.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Input Collection

**Overview:**  
This block initiates the workflow and retrieves the list of domains from a Google Sheets document, preparing them for subsequent processing.

**Nodes Involved:**  
- Manual Trigger  
- Read Domains from Google Sheets

**Node Details:**

- **Manual Trigger**  
  - *Type & Role:* Trigger node that starts the workflow manually (user clicks "Execute Workflow" or "Test Workflow").  
  - *Configuration:* No parameters needed.  
  - *Input/Output:* No input; outputs a single trigger event to start the workflow.  
  - *Version:* Standard node, no special version requirements.  
  - *Edge Cases:* None significant; user must manually start workflow or replace with a scheduled trigger for automation.

- **Read Domains from Google Sheets**  
  - *Type & Role:* Reads rows from a Google Sheets document containing domain names.  
  - *Configuration:*  
    - Document ID set to a specific Google Sheet ("BuiltWith Domain").  
    - Sheet name refers to the first tab (`gid=0`).  
    - Reads all rows; expects at least a `Domain` column (plus `row_number` for row mapping).  
    - Uses OAuth2 credentials for Google Sheets access.  
  - *Expressions:* Uses cached sheet and doc references; no dynamic expressions required for document ID or sheet name.  
  - *Input/Output:* Input from Manual Trigger; outputs each row as an item with domain data.  
  - *Version:* Uses version 4.5 of the Google Sheets node.  
  - *Edge Cases:*  
    - Missing or invalid Google Sheets credentials lead to auth error.  
    - Empty or malformed domains will propagate downstream with invalid data.  
    - Large sheets may require paging or batching if API limits are hit.

---

#### 2.2 Block 2: Tech Stack Lookup via BuiltWith API

**Overview:**  
For each domain read from Google Sheets, this block queries the BuiltWith API to retrieve technology stack information and extracts a simplified subset of data for further use.

**Nodes Involved:**  
- Fetch detail via BuiltWith (HTTP Request)  
- Extract Tech Stack Info (Code/Function node)

**Node Details:**

- **Fetch detail via BuiltWith**  
  - *Type & Role:* HTTP Request node performing a GET request to BuiltWith API to fetch tech stack info for each domain.  
  - *Configuration:*  
    - URL: `https://api.builtwith.com/v21/api.json`  
    - Query Parameters:  
      - `KEY`: Your BuiltWith API key (placeholder `"YOUR_API_KEY"` in the workflow).  
      - `LOOKUP`: Domain to analyze, dynamically set from current item’s `Domain` field (`={{ $json.Domain }}`).  
    - Sends query parameters in URL.  
  - *Input/Output:* Input from Google Sheets node; outputs JSON response from API.  
  - *Version:* HTTP Request node v4.2.  
  - *Edge Cases:*  
    - API key invalid or expired → authentication error.  
    - API rate limits exceeded → HTTP 429 or error responses.  
    - Domain not found or no data → empty or missing fields in response.  
    - Network timeouts or connectivity issues.

- **Extract Tech Stack Info (Code Node)**  
  - *Type & Role:* Function node running JavaScript code to parse the BuiltWith API response and extract a single technology entry per domain.  
  - *Configuration:*  
    - JavaScript code inspects the JSON structure returned by BuiltWith.  
    - Extracts:  
      - `Technology` (name of technology)  
      - `Category` (technology group/category)  
      - `First Detected` & `Last Detected` dates  
      - `Domain` and `URL`  
    - Only the first technology found in the first path group is extracted to keep data simple and one-to-one with domains.  
    - Returns an array with one object or empty array if no data found.  
  - *Input/Output:* Input from HTTP Request node; outputs simplified JSON object per domain.  
  - *Version:* Function node v2.  
  - *Edge Cases:*  
    - Unexpected or missing JSON structure → no extracted data; function returns empty array, effectively skipping update.  
    - Multiple technologies exist, but only one is returned — may not be suitable if users want full tech stack.  
    - Errors in JavaScript code could halt workflow or cause silent failures.

---

#### 2.3 Block 3: Output Data to Google Sheets

**Overview:**  
This block updates the original Google Sheets document with the extracted technology stack data, matching each domain row via a unique identifier.

**Nodes Involved:**  
- Update Google Sheet

**Node Details:**

- **Update Google Sheet**  
  - *Type & Role:* Google Sheets node set to update existing rows with new technology stack data.  
  - *Configuration:*  
    - Document ID and Sheet Name match those used in the read node.  
    - Operation: `update` — modifies existing rows.  
    - Matching column: `row_number` ensures the correct row is updated.  
    - Columns updated: `URL`, `Category`, `Technology`, `First Detected`, `Last Detected`.  
    - Mapping mode set to define columns explicitly.  
    - Uses OAuth2 credentials for Google Sheets access (same as reading node).  
  - *Input/Output:* Input from Function node extracting tech info; outputs updated rows confirmation.  
  - *Version:* Google Sheets node v4.5.  
  - *Edge Cases:*  
    - If `row_number` is missing or mismatched, updates could fail or affect wrong rows.  
    - Google Sheets API quota or permission errors.  
    - Data type mismatches or invalid date strings could cause update errors.

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                  | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                          |
|-----------------------------|------------------------|--------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------|
| Manual Trigger              | manualTrigger          | Starts workflow manually        | -                           | Read Domains from Google Sheets | Section 1: Input Collection – workflow starts on manual trigger to read domains from sheet                            |
| Read Domains from Google Sheets | googleSheets           | Reads domain list from Google Sheets | Manual Trigger              | Fetch detail via BuiltWith    | Section 1: Input Collection – reads domains from specified Google Sheet tab                                          |
| Fetch detail via BuiltWith  | httpRequest            | Queries BuiltWith API for tech data | Read Domains from Google Sheets | Extract Tech Stack Info       | Section 2: Tech Stack Lookup – fetches technology info from BuiltWith API for each domain                            |
| Extract Tech Stack Info     | code                   | Parses API response and extracts tech info | Fetch detail via BuiltWith   | Update Google Sheet           | Section 2: Tech Stack Lookup – extracts one technology entry per domain from API response                            |
| Update Google Sheet         | googleSheets           | Writes extracted data back to Google Sheets | Extract Tech Stack Info      | -                           | Section 3: Output to Google Sheets – updates original sheet rows with extracted tech stack data                      |
| Sticky Note                | stickyNote             | Documentation and guidance      | -                           | -                           | Section 1: Input Collection explanation                                                                            |
| Sticky Note1               | stickyNote             | Documentation and guidance      | -                           | -                           | Section 2: Tech Stack Lookup explanation                                                                            |
| Sticky Note2               | stickyNote             | Documentation and guidance      | -                           | -                           | Section 3: Output to Google Sheets explanation                                                                      |
| Sticky Note4               | stickyNote             | Workflow overview and summary   | -                           | -                           | Full workflow overview and block explanations                                                                       |
| Sticky Note9               | stickyNote             | Support and contact info        | -                           | -                           | Workflow assistance contact info and helpful links                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Manual Trigger` node:**  
   - No parameters needed.  
   - This node will start the workflow manually for testing.

3. **Add a `Google Sheets` node named "Read Domains from Google Sheets":**  
   - Operation: `Read rows`.  
   - Connect Google Sheets OAuth2 credentials (create or select existing).  
   - Set Document ID to your target Google Sheet containing domains (e.g., `1Hs9LcBpC-kaGgD_QjkkqBEJzDeAd85RW_L1fXTYEdCA`).  
   - Set Sheet Name to the tab name or ID (e.g., `gid=0` or `"Sheet1"`).  
   - Ensure your sheet has a column named `Domain` with the list of domains to query.  
   - Connect output of `Manual Trigger` → input of this node.

4. **Add an `HTTP Request` node named "Fetch detail via BuiltWith":**  
   - Method: `GET`.  
   - URL: `https://api.builtwith.com/v21/api.json`.  
   - Query Parameters:  
     - `KEY`: Your BuiltWith API key (replace `"YOUR_API_KEY"`).  
     - `LOOKUP`: Expression set to `={{ $json.Domain }}` to pass current domain.  
   - Connect output of "Read Domains from Google Sheets" → input of this node.

5. **Add a `Function` node named "Extract Tech Stack Info":**  
   - Paste the following JavaScript code:

   ```javascript
   const result = $json.Results?.[0];
   const domain = result?.Lookup || null;
   const path = result?.Result?.Paths?.[0];
   const url = path?.Url || null;

   let extracted = null;

   for (const group of path?.Groups || []) {
     const category = group.Name;
     const tech = group.Tech?.[0];

     if (tech) {
       extracted = {
         Technology: tech.Name,
         Category: category,
         "First Detected": tech.FirstDetected,
         "Last Detected": tech.LastDetected,
         Domain: domain,
         URL: url
       };
       break;
     }
   }

   return extracted ? [extracted] : [];
   ```
   - Connect output of "Fetch detail via BuiltWith" → input of this node.

6. **Add a `Google Sheets` node named "Update Google Sheet":**  
   - Operation: `Update`.  
   - Use same Google Sheets OAuth2 credentials as before.  
   - Document ID and Sheet Name must be identical to the "Read Domains from Google Sheets" node to update the same sheet.  
   - Set the matching column to `row_number` (this assumes that the `Read Domains from Google Sheets` node provides `row_number` for each row; if not, ensure your sheet includes a unique row identifier or use an alternate key).  
   - Define columns to update with expressions:  
     - `URL`: `={{ $json.URL }}`  
     - `Category`: `={{ $json.Category }}`  
     - `Technology`: `={{ $json.Technology }}`  
     - `First Detected`: `={{ $json["First Detected"] }}`  
     - `Last Detected`: `={{ $json["Last Detected"] }}`  
     - `row_number`: `={{ $('Read Domains from Google Sheets').item.json.row_number }}` to ensure correct row targeting.  
   - Connect output of "Extract Tech Stack Info" → input of this node.

7. **Save and activate the workflow.**

8. **Testing:**  
   - Populate your Google Sheet with domains under a `Domain` column.  
   - Manually trigger the workflow.  
   - Observe rows updating with technology stack data for each domain.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow assistance and support contact: Yaron@nofluff.online                                    | Contact email for questions and support                                                                        |
| YouTube channel with tips & tutorials: https://www.youtube.com/@YaronBeen/videos                 | Video resources for n8n and automation                                                                         |
| LinkedIn profile for additional insights: https://www.linkedin.com/in/yaronbeen/                 | Professional contact and updates                                                                                |
| This workflow enables scalable automation for tech stack research by integrating Google Sheets and BuiltWith API | Conceptual overview                                                                                              |
| For processing many domains, consider adding a `Wait` node or rate limiting to handle BuiltWith API limits | To avoid hitting API rate limits or overloading request volume                                                  |
| To extract all technologies per domain rather than just one, modify the function node to return multiple items | Enhances data granularity                                                                                        |
| Ensure Google Sheets API quotas and access permissions are configured correctly to avoid errors   | Credential and quota management                                                                                  |

---

**Disclaimer:**  
The provided documentation is generated from an n8n workflow automation, strictly adhering to content and data policies. All data processed is legal and public.