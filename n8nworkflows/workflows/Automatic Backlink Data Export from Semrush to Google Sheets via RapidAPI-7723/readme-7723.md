Automatic Backlink Data Export from Semrush to Google Sheets via RapidAPI

https://n8nworkflows.xyz/workflows/automatic-backlink-data-export-from-semrush-to-google-sheets-via-rapidapi-7723


# Automatic Backlink Data Export from Semrush to Google Sheets via RapidAPI

### 1. Workflow Overview

This workflow automates the process of extracting backlink data for a user-submitted website URL using the Semrush Backlink Checker API via RapidAPI, and then exporting the data into two separate Google Sheets tabs: one for backlink overview metrics and another for detailed backlink entries.

**Target Use Cases:**  
- SEO professionals or digital marketers who want to automate backlink audits and reporting.  
- Teams needing structured backlink data directly in Google Sheets for analysis or sharing.  

**Logical Blocks:**  
- **1.1 Input Reception:** Captures website URL input via a custom form trigger node.  
- **1.2 Data Retrieval:** Sends the website URL to the Semrush Backlink Checker API, handling authentication and request formatting.  
- **1.3 Data Extraction & Reformatting:** Parses the API response to separate backlink overview metrics and detailed backlink lists.  
- **1.4 Data Export:** Appends overview data and detailed backlink entries into designated sheets within a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow upon receiving a website URL from a user-submitted form.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Starts workflow on form data submission.  
    - *Configuration:*  
      - Form title: "Semrush Backlink Checker"  
      - Single required field: "website" (user inputs URL)  
      - No additional options configured.  
    - *Expressions/Variables:* Uses `$json.website` to pass the submitted URL downstream.  
    - *Input:* Triggered externally by form submission.  
    - *Output:* JSON object with `website` key containing user input.  
    - *Version:* 2.2  
    - *Potential Failures:*  
      - Missing or malformed website URL input (validation enforced by required field).  
      - Network issues preventing webhook reception.  
    - *Sub-workflow:* None.

#### 1.2 Data Retrieval

- **Overview:**  
  Sends the user-submitted website URL to the Semrush Backlink Checker API via a POST HTTP request, including required RapidAPI authentication headers.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  

  - **HTTP Request**  
    - *Type:* HTTP Request Node  
    - *Role:* Performs POST request to Semrush Backlink Checker API endpoint.  
    - *Configuration:*  
      - URL: `https://semrush-backlink-checker.p.rapidapi.com/backlink.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameters: Passes `website` parameter set to `={{ $json.website }}` (dynamic from form input).  
      - Header Parameters:  
        - `x-rapidapi-host`: `semrush-backlink-checker.p.rapidapi.com`  
        - `x-rapidapi-key`: `your key` (placeholder; requires user RapidAPI key)  
      - Sends both body and headers.  
    - *Expressions/Variables:* Uses expression to inject website URL from previous node.  
    - *Input:* Receives JSON with website URL from "On form submission".  
    - *Output:* JSON response from Semrush API containing backlink data.  
    - *Version:* 4.2  
    - *Potential Failures:*  
      - Authentication failure if RapidAPI key is invalid or missing.  
      - API rate limiting or quota exceeded errors.  
      - Network timeouts or connectivity issues.  
      - Unexpected API response format changes.  
    - *Sub-workflow:* None.

#### 1.3 Data Extraction & Reformatting

- **Overview:**  
  Splits the API response into two separate data streams: one extracting backlink overview metrics, the other extracting detailed backlink entries.

- **Nodes Involved:**  
  - Reformat 1  
  - Reformat 2

- **Node Details:**  

  - **Reformat 1**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Extracts `backlinksOverview` object from API response JSON for overview data.  
    - *Configuration:*  
      - JavaScript code: `return $input.first().json.data.semrushAPI.backlinksOverview;`  
      - This accesses nested JSON path: `data.semrushAPI.backlinksOverview`.  
    - *Input:* Receives full API response JSON from HTTP Request node.  
    - *Output:* JSON object containing backlink overview metrics.  
    - *Version:* 2  
    - *Potential Failures:*  
      - Path resolution errors if `backlinksOverview` is missing or API response format changes.  
      - Runtime JS errors if input is undefined or malformed.

  - **Reformat 2**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Extracts `backlinks` array from API response JSON for detailed backlink data.  
    - *Configuration:*  
      - JavaScript code: `return $input.first().json.data.semrushAPI.backlinks;`  
      - Accesses nested path: `data.semrushAPI.backlinks`.  
    - *Input:* Receives full API response JSON from HTTP Request node.  
    - *Output:* JSON array of backlink entries.  
    - *Version:* 2  
    - *Potential Failures:*  
      - Similar path resolution and runtime errors as Reformat 1.

#### 1.4 Data Export

- **Overview:**  
  This block appends the reformatted backlink overview data and detailed backlink entries into two separate sheets within the same Google Sheets document, using Google Sheets node with service account authentication.

- **Nodes Involved:**  
  - Backlink overview  
  - Backlinks

- **Node Details:**  

  - **Backlink overview**  
    - *Type:* Google Sheets Node  
    - *Role:* Appends backlink overview data to a specific sheet tab named "backlink overflow".  
    - *Configuration:*  
      - Operation: Append  
      - Sheet Name: "backlink overflow" (referenced by internal ID 4590546)  
      - Document ID: (empty in JSON, must be set to target Google Sheets file ID)  
      - Columns Mapping Mode: AutoMapInputData (automatically maps input fields to columns)  
      - Authentication: Service Account (credentials stored under "Google Docs account")  
    - *Input:* Receives backlink overview JSON object from "Reformat 1" node.  
    - *Output:* Appended data confirmation.  
    - *Version:* 4.6  
    - *Potential Failures:*  
      - Authentication errors if service account credentials are invalid or expired.  
      - Missing or incorrect document ID or sheet name causing write failure.  
      - Data type mismatches if input does not conform to expected schema.

  - **Backlinks**  
    - *Type:* Google Sheets Node  
    - *Role:* Appends detailed backlink entries to the "backlinks" sheet tab.  
    - *Configuration:*  
      - Operation: Append  
      - Sheet Name: "backlinks" (referenced as "gid=0")  
      - Document ID: same as above (must be set)  
      - Columns Schema: Explicitly defined fields including `targetUrl`, `sourceUrl`, `sourceTitle`, `pageAscore`, `lastSeen`, `internalNum`, `firstSeen`, `externalNum`, `anchor`, `nofollow`.  
      - Mapping Mode: AutoMapInputData  
      - Authentication: Service Account (same as above)  
    - *Input:* Receives array of backlink entries from "Reformat 2" node.  
    - *Output:* Appended data confirmation.  
    - *Version:* 4.6  
    - *Potential Failures:*  
      - Same authentication and document/sheet ID issues as "Backlink overview" node.  
      - Data mismatch if input fields are missing or malformed.  
      - API rate limits or quota issues with Google Sheets API.

---

### 3. Summary Table

| Node Name          | Node Type          | Functional Role                                   | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                      |
|--------------------|--------------------|-------------------------------------------------|------------------------|----------------------|------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger       | Receives website URL input via form submission  | —                      | HTTP Request         | **On form submission** – Starts the workflow when a user submits a website URL through a custom form. |
| HTTP Request       | HTTP Request       | Sends website URL to Semrush Backlink Checker API | On form submission      | Reformat 1, Reformat 2 | **HTTP Request** – Sends the submitted URL to Semrush Backlink Checker API using POST with headers and body data. |
| Reformat 1          | Code               | Extracts backlink overview metrics from API response | HTTP Request           | Backlink overview    | **Reformat 1** – Extracts the `backlinksOverview` object from the API response JSON.             |
| Reformat 2          | Code               | Extracts detailed backlink list from API response | HTTP Request           | Backlinks            | **Reformat 2** – Extracts the `backlinks` list (detailed data) from the API response JSON.       |
| Backlink overview   | Google Sheets      | Appends backlink overview to Google Sheet       | Reformat 1              | —                    | **Backlink overview** – Appends backlink overview metrics to the “backlink overflow” sheet in a specified Google Sheets document. |
| Backlinks           | Google Sheets      | Appends detailed backlink entries to Google Sheet | Reformat 2              | —                    | **Backlinks** – Appends individual backlink entries (e.g. source URL, anchor, score) to the “backlinks” sheet in the same Google Sheet. |
| Sticky Note         | Sticky Note        | Documentation and summary                         | —                      | —                    | # 🚀 Semrush Backlink Checker Automation with n8n and Google Sheets... (full content in node)    |
| Sticky Note1        | Sticky Note        | Explains On form submission node                  | —                      | —                    | **On form submission** – Starts the workflow when a user submits a website URL through a custom form. |
| Sticky Note2        | Sticky Note        | Explains HTTP Request node                         | —                      | —                    | **HTTP Request** – Sends the submitted URL to Semrush Backlink Checker API using POST with headers and body data. |
| Sticky Note3        | Sticky Note        | Explains Reformat 1 node                           | —                      | —                    | **Reformat 1** – Extracts the `backlinksOverview` object from the API response JSON.             |
| Sticky Note4        | Sticky Note        | Explains Reformat 2 node                           | —                      | —                    | **Reformat 2** – Extracts the `backlinks` list (detailed data) from the API response JSON.       |
| Sticky Note5        | Sticky Note        | Explains Backlink overview node                    | —                      | —                    | **Backlink overview** – Appends backlink overview metrics to the “backlink overflow” sheet in a specified Google Sheets document. |
| Sticky Note6        | Sticky Note        | Explains Backlinks node                            | —                      | —                    | **Backlinks** – Appends individual backlink entries (e.g. source URL, anchor, score) to the “backlinks” sheet in the same Google Sheet. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node:**  
   - Type: Form Trigger  
   - Configure form title as "Semrush Backlink Checker".  
   - Add one required field named "website" for URL input.  
   - This node will trigger the workflow when the form is submitted.

2. **Create "HTTP Request" node:**  
   - Type: HTTP Request  
   - Connect input from "On form submission" node.  
   - Set HTTP method to POST.  
   - Set URL to `https://semrush-backlink-checker.p.rapidapi.com/backlink.php`.  
   - Set content type to `multipart/form-data`.  
   - Add body parameter:  
     - Name: `website`  
     - Value: Expression `{{$json["website"]}}`  
   - Add header parameters:  
     - `x-rapidapi-host`: `semrush-backlink-checker.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key string (replace placeholder).  
   - Enable sending both headers and body.

3. **Create "Reformat 1" node:**  
   - Type: Code  
   - Connect input from "HTTP Request" node.  
   - Use JavaScript code:  
     ```js
     return $input.first().json.data.semrushAPI.backlinksOverview;
     ```  
   - This extracts the backlink overview data object.

4. **Create "Reformat 2" node:**  
   - Type: Code  
   - Connect input from "HTTP Request" node.  
   - Use JavaScript code:  
     ```js
     return $input.first().json.data.semrushAPI.backlinks;
     ```  
   - This extracts the detailed backlinks array.

5. **Create "Backlink overview" Google Sheets node:**  
   - Type: Google Sheets  
   - Connect input from "Reformat 1" node.  
   - Operation: Append  
   - Set Document ID to your target Google Sheets file ID.  
   - Set Sheet Name to "backlink overflow" (or use respective sheet ID 4590546).  
   - Enable AutoMapInputData columns mapping.  
   - Authenticate using Google API Service Account credentials.  
   - Ensure the service account has write access to the Google Sheet.

6. **Create "Backlinks" Google Sheets node:**  
   - Type: Google Sheets  
   - Connect input from "Reformat 2" node.  
   - Operation: Append  
   - Set Document ID to same Google Sheets file ID as above.  
   - Set Sheet Name to "backlinks" (sheet ID or "gid=0").  
   - Define columns schema exactly with fields:  
     `targetUrl`, `sourceUrl`, `sourceTitle`, `pageAscore`, `lastSeen`, `internalNum`, `firstSeen`, `externalNum`, `anchor`, `nofollow`.  
   - Use AutoMapInputData mapping.  
   - Authenticate with same Google API Service Account credentials.

7. **Test the workflow:**  
   - Open the form URL generated by the "On form submission" node.  
   - Submit a valid website URL.  
   - Verify data populates correctly in both Google Sheets tabs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| 🚀 Semrush Backlink Checker Automation with n8n and Google Sheets enables streamlined backlink audits and reporting workflows. | Overview and purpose of the workflow, as described in the main Sticky Note.                                      |
| The Semrush Backlink Checker API is accessed via RapidAPI and requires a valid API key for authentication.                    | API authentication details and potential failure points in HTTP Request node.                                    |
| Google Sheets nodes require a service account with write permissions to the target spreadsheet.                               | Credential setup details—ensure proper Google Cloud IAM permissions and share the sheet with the service account. |
| Form Trigger node provides an easy way to collect website URLs without external form management tools.                        | Simplifies user input for SEO backlink lookup workflows.                                                         |

---

**Disclaimer:** The provided text is generated exclusively from an automated n8n workflow export. It complies strictly with content policies and contains no illegal or protected material. All handled data is legal and public.