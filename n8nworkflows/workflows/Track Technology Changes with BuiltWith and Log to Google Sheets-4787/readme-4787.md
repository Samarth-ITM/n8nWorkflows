Track Technology Changes with BuiltWith and Log to Google Sheets

https://n8nworkflows.xyz/workflows/track-technology-changes-with-builtwith-and-log-to-google-sheets-4787


# Track Technology Changes with BuiltWith and Log to Google Sheets

### 1. Workflow Overview

This workflow is designed to track technology usage across websites by querying the BuiltWith API for sites using a specified technology and logging the results into a Google Sheet. It is useful for lead generation, competitor analysis, marketing research, and technology monitoring.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Manual initiation of the workflow and specification of the target technology to search for.
- **1.2 API Data Retrieval and Processing:** Querying the BuiltWith API for websites using the specified technology, then parsing and formatting the response data.
- **1.3 Data Logging:** Appending the cleaned data into a Google Sheets document for record keeping and further analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block allows the user to manually trigger the workflow and specify the technology to be tracked (e.g., “Shopify”). It prepares the input data that drives the API request.

**Nodes Involved:**  
- Manual Trigger  
- Set Technology

**Node Details:**

- **Manual Trigger**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow on user demand rather than on a schedule or event.  
  - *Configuration:* No parameters; simply waits for manual execution.  
  - *Inputs:* None  
  - *Outputs:* One output to “Set Technology” node.  
  - *Edge Cases:* None typical; user must manually trigger.  
  - *Version:* n8n v1 or later supports this node.

- **Set Technology**  
  - *Type:* Set node  
  - *Role:* Defines a static variable `Technology` with the technology name to track, e.g., “Shopify”.  
  - *Configuration:* Assigns a string field named `Technology` with value “Shopify”. This makes the workflow flexible to different technologies by editing this one node.  
  - *Expressions:* The Technology value is set statically; downstream nodes reference it via `{{$json.Technology}}`.  
  - *Inputs:* From “Manual Trigger”  
  - *Outputs:* To “Fetch BuiltWith Data”  
  - *Edge Cases:* If the technology name is misspelled or empty, the API query might return no results or errors.  

---

#### 1.2 API Data Retrieval and Processing

**Overview:**  
This block sends a GET request to the BuiltWith API, querying for websites using the specified technology. It then processes the JSON response to extract key fields into a simplified structure.

**Nodes Involved:**  
- Fetch BuiltWith Data  
- Extract Site Info

**Node Details:**

- **Fetch BuiltWith Data**  
  - *Type:* HTTP Request  
  - *Role:* Calls the BuiltWith API endpoint to retrieve websites using the specified technology.  
  - *Configuration:*  
    - Method: GET  
    - URL: `https://api.builtwith.com/v20/api.json`  
    - Query parameters:  
      - `KEY`: API key (replace `YOUR_API_KEY` with actual key)  
      - `TECH`: Set dynamically from `{{$json.Technology}}`  
      - `START`: 0 (pagination start index)  
    - No authentication within the node; API key is passed as query param.  
  - *Inputs:* From “Set Technology”  
  - *Outputs:* To “Extract Site Info”  
  - *Edge Cases:*  
    - Invalid or missing API key causes auth errors.  
    - Incorrect technology names may yield empty results.  
    - API rate limits or timeouts possible.  
  - *Version:* Requires n8n supporting HTTP Request v4.2 or later.

- **Extract Site Info**  
  - *Type:* Code (Function) node  
  - *Role:* Parses the BuiltWith API response and maps each website’s data into a simplified JSON with keys: `domain`, `technology`, `firstIndexed`, and `vertical`.  
  - *Configuration:* JavaScript code processes the `Results` array from the API response, extracting fields such as `ResultPath` (domain), `FirstIndexed` (date), and `Vertical` (industry). It also assigns the technology searched for from input data.  
  - *Key expressions:* Accesses `$json.Results` and `$json.technologies` fields.  
  - *Inputs:* From “Fetch BuiltWith Data”  
  - *Outputs:* Array of JSON objects, each representing one site, sent to next node.  
  - *Edge Cases:* If `Results` is missing or empty, outputs an empty array. Defensive fallback for missing fields to empty strings.  
  - *Version:* n8n v2 or later recommended for code node stability.

---

#### 1.3 Data Logging

**Overview:**  
This final block appends the formatted website data into a Google Sheets document, enabling long-term storage and further use such as lead tracking or analytics.

**Nodes Involved:**  
- Log to Google Sheet

**Node Details:**

- **Log to Google Sheet**  
  - *Type:* Google Sheets node (Append operation)  
  - *Role:* Inserts each site’s data as a new row in a specified Google Sheet tab.  
  - *Configuration:*  
    - Operation: Append  
    - Document ID: Spreadsheet ID `1lDu3HvxcpA1nRDJq6Ge-OrMO0v592UfLBIjt3Ylc3Mo`  
    - Sheet Name: `gid=0` (Sheet1)  
    - Columns mapped explicitly:  
      - Domain ← `{{$json.domain}}`  
      - Vertical ← `{{$json.vertical}}`  
      - Technology ← `{{$json.technology}}`  
      - First Indexed ← `{{$json.firstIndexed}}`  
    - Mapping Mode: Define below, no automatic matching.  
  - *Credentials:* Google Sheets OAuth2 (previously authenticated)  
  - *Inputs:* From “Extract Site Info”  
  - *Outputs:* None (end of workflow)  
  - *Edge Cases:*  
    - Authentication failures if OAuth token expires.  
    - Sheet or document access permission denied errors.  
    - Data type mismatches if Google Sheets schema changes.  
  - *Version:* Compatible with n8n Google Sheets node v4.5 or higher.

---

### 3. Summary Table

| Node Name            | Node Type       | Functional Role                        | Input Node(s)      | Output Node(s)      | Sticky Note                                                                                                    |
|----------------------|-----------------|-------------------------------------|--------------------|---------------------|---------------------------------------------------------------------------------------------------------------|
| Manual Trigger       | Manual Trigger  | Manual start of workflow             | -                  | Set Technology      | 🧩 SECTION 1: Manual Start + Set Target Technology. Purpose: manual run & technology selection.                 |
| Set Technology       | Set             | Define technology to search (e.g. Shopify) | Manual Trigger     | Fetch BuiltWith Data | 🧩 SECTION 1: Manual Start + Set Target Technology.                                                             |
| Fetch BuiltWith Data | HTTP Request    | Query BuiltWith API for websites     | Set Technology     | Extract Site Info    | 🔗 SECTION 2: Fetch + Process BuiltWith Data. Purpose: API call and JSON processing.                            |
| Extract Site Info    | Code (Function) | Parse and format API response        | Fetch BuiltWith Data | Log to Google Sheet  | 🔗 SECTION 2: Fetch + Process BuiltWith Data.                                                                   |
| Log to Google Sheet  | Google Sheets   | Append parsed data to Google Sheet   | Extract Site Info  | -                   | 📊 SECTION 3: Log to Google Sheets. Purpose: store results for lead gen, tracking, campaigns.                   |
| Sticky Note          | Sticky Note     | Documentation and explanation        | -                  | -                   | Contains detailed block explanations and tips (multiple nodes).                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Add a “Manual Trigger” node with default settings. No parameters needed. This will manually start the workflow.

2. **Create Set node named “Set Technology”**  
   - Add a “Set” node.  
   - Add a new field named `Technology` (string type).  
   - Set its value to the technology you want to track, e.g., `Shopify`.  
   - Connect “Manual Trigger” → “Set Technology”.

3. **Create HTTP Request node named “Fetch BuiltWith Data”**  
   - Add an “HTTP Request” node.  
   - Set Method to `GET`.  
   - Set URL to `https://api.builtwith.com/v20/api.json`.  
   - Add Query Parameters:  
     - `KEY`: Your BuiltWith API Key (replace placeholder).  
     - `TECH`: Use expression referencing `{{$json.Technology}}` from previous node.  
     - `START`: `0` (to start pagination).  
   - Connect “Set Technology” → “Fetch BuiltWith Data”.  
   - Note: Ensure your BuiltWith API key is valid and has permissions.

4. **Create Code node named “Extract Site Info”**  
   - Add a “Function” node.  
   - Paste the following JavaScript code:  
     ```js
     const techSearched = $json["technologies"] || $json.Technology || "Unknown";
     const results = $json.Results || [];
     
     return results.map(site => {
       return {
         json: {
           domain: site.ResultPath || "",
           technology: techSearched,
           firstIndexed: site.FirstIndexed || "",
           vertical: site.Vertical || ""
         }
       };
     });
     ```  
   - Connect “Fetch BuiltWith Data” → “Extract Site Info”.

5. **Create Google Sheets node named “Log to Google Sheet”**  
   - Add a “Google Sheets” node.  
   - Select Operation: `Append`.  
   - Enter the Google Sheet Document ID (e.g., `1lDu3HvxcpA1nRDJq6Ge-OrMO0v592UfLBIjt3Ylc3Mo`).  
   - Select Sheet Name or specify `gid=0` for the first sheet.  
   - Set Columns mapping explicitly:  
     - Domain ← `{{$json.domain}}`  
     - Vertical ← `{{$json.vertical}}`  
     - Technology ← `{{$json.technology}}`  
     - First Indexed ← `{{$json.firstIndexed}}`  
   - Connect “Extract Site Info” → “Log to Google Sheet”.  
   - Configure Google Sheets OAuth2 credentials with appropriate permissions (read/write access to the spreadsheet).

6. **Optional: Add Sticky Notes for documentation**  
   - Add sticky notes at relevant places explaining each section for easier maintenance.

7. **Test the workflow**  
   - Run manually, check logs and Google Sheet for appended rows.  
   - Handle errors such as invalid API keys or permissions by updating credentials or input data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| For support or questions, contact: Yaron@nofluff.online                                                                                                  | Workflow Assistance Sticky Note                                                                |
| YouTube channel with related n8n tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                                           | Support and learning resource                                                                   |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                                                                | Professional profile for workflow author                                                        |
| BuiltWith API documentation and usage tips: https://builtwith.com/                                                                                       | API reference for extending or modifying the workflow                                           |
| Google Sheets OAuth2 setup required for appending data: https://developers.google.com/sheets/api/guides/authorizing                                          | Credential setup instructions for Google Sheets node                                            |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, a no-code automation platform. It complies fully with content policies and contains no illegal or offensive elements. All data processed is legal and publicly accessible.