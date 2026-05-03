Check Domain Authority Metrics in Bulk and Log to Google Sheets with RapidAPI

https://n8nworkflows.xyz/workflows/check-domain-authority-metrics-in-bulk-and-log-to-google-sheets-with-rapidapi-8249


# Check Domain Authority Metrics in Bulk and Log to Google Sheets with RapidAPI

### 1. Workflow Overview

This n8n workflow automates the process of checking Domain Authority (DA) and Page Authority (PA) metrics for multiple domains in bulk, then logs the results into a Google Sheet for easy access, tracking, and reporting. It is designed for SEO professionals, agencies, bloggers, or teams needing regular domain authority checks.

**Logical blocks:**

- **1.1 Input Reception:** Captures user input of comma-separated domain names via a public form.
- **1.2 API Request:** Sends the domains list to the Bulk DA PA Checker API hosted on RapidAPI using a POST request.
- **1.3 Response Processing:** Reformats and extracts DA/PA data from the API response, flattening it for Google Sheets.
- **1.4 Data Logging:** Appends each domain's metrics as rows into a designated Google Sheet using service account authentication.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  This block initiates the workflow by providing a user-friendly web form to submit a list of domains (comma-separated) for DA/PA checking.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note1

- **Node Details:**

  - **On form submission**  
    - *Type & Role:* Form Trigger node; triggers workflow on form submission.  
    - *Configuration:*  
      - Webhook ID assigned for public access.  
      - Form titled "Bulk DA PA Checker".  
      - Single required field: "Domains" (placeholder: "Comma separated domains").  
      - No additional options enabled.  
    - *Expressions/Variables:* Input value captured as `$json.Domains`.  
    - *Connections:* Output → "Check DA PA Bulk" node.  
    - *Potential Failures:* Form not accessible if webhook misconfigured; missing required field leads to no trigger; input format issues downstream if domains not comma-separated correctly.  
    - *Version:* 2.2  

  - **Sticky Note1**  
    - *Role:* Documentation note describing the form's purpose.  
    - *Content:* "Presents a public-facing form where users enter a comma-separated list of domains."  

#### Block 1.2: API Request

- **Overview:**  
  Sends the submitted domain list to the Bulk DA PA Checker API via RapidAPI to retrieve DA and PA metrics.

- **Nodes Involved:**  
  - Check DA PA Bulk  
  - Sticky Note2

- **Node Details:**

  - **Check DA PA Bulk**  
    - *Type & Role:* HTTP Request node; performs POST request to RapidAPI endpoint.  
    - *Configuration:*  
      - URL: `https://bulk-da-pa-checker2.p.rapidapi.com/bulk-dapa.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameters: `domains` set dynamically to the form input via `={{ $json.Domains }}` expression.  
      - Headers:  
        - `x-rapidapi-host`: `bulk-da-pa-checker2.p.rapidapi.co` (note: possible typo, should confirm).  
        - `x-rapidapi-key`: placeholder `"your key"`; requires valid RapidAPI key credential.  
      - Sends headers and body as required by API.  
    - *Connections:* Output → "Re Format" node.  
    - *Potential Failures:*  
      - Authentication errors if API key is invalid or missing.  
      - Timeout or network errors.  
      - API response format changes.  
      - Possible typo in `x-rapidapi-host` header may cause request failure.  
    - *Version:* 4.2  

  - **Sticky Note2**  
    - *Role:* Documentation describing this node's function.  
    - *Content:* "Sends the list to the [Bulk DA PA Checker API](https://rapidapi.com/skdeveloper/api/bulk-da-pa-checker2) using a POST request."  

#### Block 1.3: Response Processing

- **Overview:**  
  Processes the API JSON response by iterating over each domain's data, flattening the structure, and adding the domain name explicitly to each record for easy logging.

- **Nodes Involved:**  
  - Re Format  
  - Sticky Note3

- **Node Details:**

  - **Re Format**  
    - *Type & Role:* Code node; executes JavaScript to parse and restructure API response.  
    - *Configuration:*  
      - JavaScript code extracts the `data` property from the first item in input.  
      - Iterates each domain key and its associated metrics object.  
      - Adds domain name as explicit field `domain` in each row.  
      - Returns an array of JSON objects, one per domain, suitable for Google Sheets input.  
    - *Key Expressions:* Uses `$input.first().json.data` to access API response data.  
    - *Connections:* Output → "Append In Google Sheets".  
    - *Potential Failures:*  
      - If API response structure changes (e.g., missing `data` or domains), code may throw errors or return empty results.  
      - Malformed JSON or unexpected data types.  
    - *Version:* 2  

  - **Sticky Note3**  
    - *Role:* Documentation describing code node's processing.  
    - *Content:* "Loops through the API response, extracts each domain's data, and flattens the fields for easy row-by-row logging."  

#### Block 1.4: Data Logging

- **Overview:**  
  Appends each processed domain authority record as a row into a specific Google Sheet for persistent storage and later analysis.

- **Nodes Involved:**  
  - Append In Google Sheets  
  - Sticky Note4

- **Node Details:**

  - **Append In Google Sheets**  
    - *Type & Role:* Google Sheets node; appends rows to a spreadsheet using service account authentication.  
    - *Configuration:*  
      - Operation: Append  
      - Document ID: (empty in JSON; must be set to target Google Sheet ID)  
      - Sheet Name: `gid=0` (Sheet1)  
      - Columns defined explicitly: `success`, `courses`, `totalItems`, `currentPage`, `totalPages` (note: these columns do not align with domain authority data fields, suggesting configuration needs updating or dynamic mode is not used)  
      - Authentication: Service Account with credentials named "Google Docs account".  
      - Convert types disabled; data passed as-is.  
    - *Connections:* Input from "Re Format" node.  
    - *Potential Failures:*  
      - Authentication errors if service account credentials invalid or insufficient permissions.  
      - Document ID missing or incorrect will cause failure to append.  
      - Mismatched columns vs data fields may cause data misalignment or loss.  
      - API rate limits or Google Sheets API errors.  
    - *Version:* 4.6  

  - **Sticky Note4**  
    - *Role:* Documentation describing Google Sheets storage.  
    - *Content:* "Saves the full DA/PA report for each domain into a Google Sheet for tracking, reporting, or export."  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                                | Input Node(s)     | Output Node(s)           | Sticky Note                                                                                                                       |
|---------------------|---------------------|-----------------------------------------------|-------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger        | Receives comma-separated domains input        | -                 | Check DA PA Bulk          | Presents a public-facing form where users enter a comma-separated list of domains.                                               |
| Check DA PA Bulk     | HTTP Request        | Sends domains list to Bulk DA PA Checker API  | On form submission| Re Format                 | Sends the list to the [Bulk DA PA Checker API](https://rapidapi.com/skdeveloper/api/bulk-da-pa-checker2) using a POST request.   |
| Re Format            | Code                | Processes and flattens API response data      | Check DA PA Bulk  | Append In Google Sheets   | Loops through the API response, extracts each domain's data, and flattens the fields for easy row-by-row logging.               |
| Append In Google Sheets | Google Sheets      | Appends domain authority data to Google Sheet | Re Format        | -                        | Saves the full DA/PA report for each domain into a Google Sheet for tracking, reporting, or export.                               |
| Sticky Note          | Sticky Note         | Documentation                                 | -                 | -                        | # 🌐 Bulk Domain Authority (DA/PA) Checker - Overview and use case detailed.                                                     |
| Sticky Note1         | Sticky Note         | Documentation                                 | -                 | -                        | Presents a public-facing form where users enter a comma-separated list of domains.                                               |
| Sticky Note2         | Sticky Note         | Documentation                                 | -                 | -                        | Sends the list to the [Bulk DA PA Checker API](https://rapidapi.com/skdeveloper/api/bulk-da-pa-checker2) using a POST request.   |
| Sticky Note3         | Sticky Note         | Documentation                                 | -                 | -                        | Loops through the API response, extracts each domain's data, and flattens the fields for easy row-by-row logging.               |
| Sticky Note4         | Sticky Note         | Documentation                                 | -                 | -                        | Saves the full DA/PA report for each domain into a Google Sheet for tracking, reporting, or export.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Node Type: Form Trigger  
   - Configure:  
     - Set form title to "Bulk DA PA Checker".  
     - Add one required field labeled "Domains" with placeholder "Comma separated domains".  
     - Obtain and note the webhook URL for external access.  

2. **Create the HTTP Request Node (Bulk DA PA Checker API)**  
   - Node Type: HTTP Request  
   - Configure:  
     - URL: `https://bulk-da-pa-checker2.p.rapidapi.com/bulk-dapa.php`  
     - Method: POST  
     - Content-Type: multipart/form-data  
     - Body Parameters: Add `domains` parameter with value set to expression `{{$json.Domains}}` to pass form input.  
     - Header Parameters:  
       - `x-rapidapi-host`: `bulk-da-pa-checker2.p.rapidapi.co` (verify correct hostname).  
       - `x-rapidapi-key`: Insert your valid RapidAPI key here.  
     - Enable sending headers and body.  
   - Connect output of Form Trigger node to this HTTP Request node.  

3. **Create the Code Node to Reformat API Response**  
   - Node Type: Code  
   - Configure:  
     - Use JavaScript code:  
       ```javascript
       const raw = $input.first().json.data;
       const results = [];
       for (const domain in raw) {
         const row = raw[domain];
         row.domain = domain;
         results.push({ json: row });
       }
       return results;
       ```  
   - Connect output of HTTP Request node to this Code node.  

4. **Create the Google Sheets Node to Append Data**  
   - Node Type: Google Sheets  
   - Configure:  
     - Operation: Append  
     - Document ID: Set to your Google Sheet ID where data will be stored.  
     - Sheet Name: Use `Sheet1` or specify sheet by gid (e.g., `gid=0`).  
     - Columns: Ideally set to match the keys in the reformatted data (including `domain` and DA/PA fields). Adjust columns accordingly.  
     - Authentication: Use a Google Service Account credential with access to your Google Sheets document.  
     - Disable "Convert Fields to String" unless needed.  
   - Connect output of Code node to this Google Sheets node.  

5. **Add Sticky Notes For Documentation (Optional but Recommended)**  
   - Add sticky notes near each node describing its function for clarity and maintenance. Use the content from the sticky notes in the original workflow.  

6. **Test the Workflow**  
   - Submit a comma-separated domain list via the form webhook URL.  
   - Confirm that the API request succeeds and returns expected data.  
   - Check that the code node parses data without errors.  
   - Verify rows are appended correctly in Google Sheet with correct fields.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                      | Context or Link                                                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses the [Bulk DA PA Checker API](https://rapidapi.com/skdeveloper/api/bulk-da-pa-checker2) on RapidAPI to retrieve domain metrics. | API documentation and subscription at RapidAPI: https://rapidapi.com/skdeveloper/api/bulk-da-pa-checker2                                                              |
| Use Google Service Account credentials with appropriate Sheets API permissions for automated appending without user interaction.                  | Google Cloud Console IAM & Admin: https://console.cloud.google.com/iam-admin/serviceaccounts                                                                             |
| The form trigger exposes a webhook URL to receive domain lists; secure and restrict access as needed for production.                               | n8n webhook & form trigger docs: https://docs.n8n.io/nodes/n8n-nodes-base.formtrigger/                                                                                 |
| Workflow ideal for SEO teams, agencies, and bloggers automating domain authority monitoring and reporting.                                         | Sticky note content in workflow includes use cases and scenario descriptions.                                                                                           |

---

**Disclaimer:** The text provided derives exclusively from an automated n8n workflow. All data and API usage comply with legal and content policies.