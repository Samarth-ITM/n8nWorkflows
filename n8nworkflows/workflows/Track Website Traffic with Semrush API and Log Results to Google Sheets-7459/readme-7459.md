Track Website Traffic with Semrush API and Log Results to Google Sheets

https://n8nworkflows.xyz/workflows/track-website-traffic-with-semrush-api-and-log-results-to-google-sheets-7459


# Track Website Traffic with Semrush API and Log Results to Google Sheets

### 1. Workflow Overview

This workflow automates the process of tracking website traffic by integrating user-submitted website URLs with the Semrush API and logging the resulting traffic data into a Google Sheets document. It targets use cases involving SEO analysts, digital marketers, or webmasters who want to collect, monitor, and store website traffic metrics efficiently.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Captures website URLs submitted by users through an n8n form.
- **1.2 API Request:** Sends a request to the Semrush API (via RapidAPI) to retrieve traffic data for the submitted website.
- **1.3 Data Processing:** Extracts and reformats the relevant traffic metrics from the API response for easier downstream usage.
- **1.4 Data Storage:** Appends the cleaned traffic data into a specified Google Sheets document for record-keeping and analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for form submissions containing a website URL. It triggers the workflow when a user submits the form.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point that captures user input from a custom form.  
    - Configuration:  
      - Form Title: "website traffic checker"  
      - Form Description: "website traffic checker"  
      - Form Fields: Single required input field labeled "website"  
    - Key Variables: The submitted JSON includes a `website` field used downstream.  
    - Input Connections: None (trigger node)  
    - Output Connections: Connected to "Check webTraffic" node  
    - Version: 2.2  
    - Edge Cases:  
      - Missing or empty website URL (handled by required field setting)  
      - Malformed URL input (not validated here, may cause API errors downstream)  
    - Notes: Triggers the entire workflow upon user submission.

#### 1.2 API Request

- **Overview:**  
  Sends a POST request to the Semrush API via RapidAPI to fetch website traffic statistics for the provided website URL.

- **Nodes Involved:**  
  - Check webTraffic

- **Node Details:**

  - **Check webTraffic**  
    - Type: HTTP Request  
    - Role: Queries the Semrush API to retrieve traffic data.  
    - Configuration:  
      - Method: POST  
      - URL: `https://website-traffic-checker-semrush.p.rapidapi.com/website-traffic.php`  
      - Content-Type: multipart/form-data  
      - Body Parameter: `website` set dynamically from form input (`{{$json.website}}`)  
      - Headers include:  
        - `x-rapidapi-host`: "website-traffic-checker-semrush.p.rapidapi.com"  
        - `x-rapidapi-key`: Placeholder ("your key") requiring user API key setup  
    - Input Connections: Receives input from "On form submission"  
    - Output Connections: Connected to "Re format output" node  
    - Version: 4.2  
    - Edge Cases:  
      - API authentication failure if key is invalid or missing  
      - API rate limits or downtime leading to request failure or timeout  
      - Unexpected or missing data in response  
    - Notes: Requires valid RapidAPI credentials for Semrush API access.

#### 1.3 Data Processing

- **Overview:**  
  Processes the raw API response to extract the first element of the traffic summary data, simplifying it for storage.

- **Nodes Involved:**  
  - Re format output

- **Node Details:**

  - **Re format output**  
    - Type: Code (JavaScript)  
    - Role: Extracts relevant traffic data from nested API response JSON.  
    - Configuration:  
      - JavaScript code: `return $input.first().json.data.semrushAPI.trafficSummary[0]`  
    - Input Connections: Receives API response from "Check webTraffic"  
    - Output Connections: Connected to "Google Sheets"  
    - Version: 2  
    - Edge Cases:  
      - If the API response structure changes or `trafficSummary` is empty, this will cause errors or produce undefined output  
      - Missing or malformed data leads to failures in extraction  
    - Notes: Simplifies the JSON structure for easier mapping in the next step.

#### 1.4 Data Storage

- **Overview:**  
  Appends the processed traffic data into a Google Sheets document, organizing it by predefined columns.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends incoming data as a new row into a specified Google Sheet.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Configured via credential reference (named "Seo n8n")  
      - Sheet Name: Uses `gid=0` (first sheet/tab)  
      - Columns mapped automatically from input JSON fields:  
        - searchOrganic, pagesPerVisit, target, visits, users, timeOnSite, bounceRate, displayDate  
      - Authentication: Service account credentials for Google API  
    - Input Connections: Receives cleaned data from "Re format output"  
    - Output Connections: None (end node)  
    - Version: 4.6  
    - Edge Cases:  
      - Authentication failures or lack of permission on Google Sheets API  
      - Missing or mismatched fields causing append failures  
      - Google Sheets API limits or quota exhaustion  
    - Notes: Stores data for long-term tracking and analysis.

---

### 3. Summary Table

| Node Name          | Node Type         | Functional Role                     | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                          |
|--------------------|-------------------|-----------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------------|
| On form submission | Form Trigger      | Captures website URL input         | None                  | Check webTraffic      | ### 1. **On form submission**  \nCaptures the website URL submitted by the user through a form.  \nTriggers the flow to start when the form is filled and submitted. |
| Check webTraffic    | HTTP Request      | Requests traffic data from Semrush | On form submission    | Re format output      | ### 2. **Check webTraffic**  \nSends a request to the Semrush API to gather website traffic data.  \nUses the URL submitted by the user to fetch traffic statistics.    |
| Re format output    | Code (JavaScript) | Extracts relevant traffic data     | Check webTraffic      | Google Sheets         | ### 3. **Re format output**  \nExtracts and reformats the relevant traffic data from the API response.  \nCleans the raw data for easier processing and usage.                 |
| Google Sheets       | Google Sheets     | Appends data to Google Sheets      | Re format output      | None                  | ### 4. **Google Sheets**  \nAppends the formatted traffic data into a designated Google Sheets document.  \nStores the data for later analysis and reporting.                 |
| Sticky Note         | Sticky Note       | Documentation                     | None                  | None                  | # SEO Website Traffic Checker\n\n## Flow Description:\nThis workflow collects website traffic data based on a submitted website URL and appends the data to a Google Sheet.\n\n---\n\n## Node-by-Node Explanation\n\n### 1. **On form submission**  \nCaptures the website URL submitted by the user through a form.\n\n### 2. **Check webTraffic**  \nSends a request to the Semrush API to gather website traffic data.\n\n### 3. **Re format output**  \nExtracts and reformats the relevant traffic data from the API response.\n\n### 4. **Google Sheets** \nAppends the formatted traffic data into a designated Google Sheets document. |
| Sticky Note1        | Sticky Note       | Documentation                     | None                  | None                  | ### 1. **On form submission**  \nCaptures the website URL submitted by the user through a form.  \nTriggers the flow to start when the form is filled and submitted.          |
| Sticky Note2        | Sticky Note       | Documentation                     | None                  | None                  | ### 2. **Check webTraffic**  \nSends a request to the Semrush API to gather website traffic data.  \nUses the URL submitted by the user to fetch traffic statistics.          |
| Sticky Note3        | Sticky Note       | Documentation                     | None                  | None                  | ### 3. **Re format output**  \nExtracts and reformats the relevant traffic data from the API response.  \nCleans the raw data for easier processing and usage.               |
| Sticky Note4        | Sticky Note       | Documentation                     | None                  | None                  | ### 4. **Google Sheets**  \nAppends the formatted traffic data into a designated Google Sheets document.  \nStores the data for later analysis and reporting.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure:  
     - Form Title: "website traffic checker"  
     - Form Description: "website traffic checker"  
     - Add one form field: Label = "website", Required = true  
   - Save and position as the workflow entry trigger.

2. **Create "Check webTraffic" node**  
   - Type: HTTP Request  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://website-traffic-checker-semrush.p.rapidapi.com/website-traffic.php`  
     - Content-Type: multipart/form-data  
     - Body Parameters: Add parameter named `website`, set value to expression `{{$json["website"]}}` (dynamic from form input)  
     - Headers Parameters:  
       - Add `x-rapidapi-host` with value `website-traffic-checker-semrush.p.rapidapi.com`  
       - Add `x-rapidapi-key` with your RapidAPI key for Semrush API (must be set up beforehand)  
     - Enable sending body and headers  
   - Connect "On form submission" node output to this node input.

3. **Create "Re format output" node**  
   - Type: Code (JavaScript)  
   - Configure:  
     - JavaScript code:  
       ```javascript
       return $input.first().json.data.semrushAPI.trafficSummary[0];
       ```  
   - Connect "Check webTraffic" node output to this node input.

4. **Create "Google Sheets" node**  
   - Type: Google Sheets  
   - Configure:  
     - Operation: Append  
     - Document ID: Set to your target Google Sheets document ID (or select from credentials if available)  
     - Sheet Name: Use the first sheet/tab (`gid=0`)  
     - Column Mapping: Enable automatic mapping from input JSON fields: `searchOrganic`, `pagesPerVisit`, `target`, `visits`, `users`, `timeOnSite`, `bounceRate`, `displayDate`  
     - Authentication: Use Google Service Account credentials with write permissions on the target sheet  
   - Connect "Re format output" node output to this node input.

5. **Credentials Setup**  
   - Setup Google API OAuth2 or Service Account credentials with sufficient permissions for Google Sheets access.  
   - Setup RapidAPI credentials with a valid API key for the Semrush API.

6. **Test the workflow**  
   - Submit the form with a valid website URL.  
   - Ensure the API returns data and data is appended correctly in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                    |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow collects website traffic data based on a submitted website URL and appends the data to a Google Sheet. | Workflow purpose summary (Sticky Note content)    |
| Requires valid RapidAPI key for Semrush API interaction.                                         | API key must be obtained at https://rapidapi.com/ |
| Google Sheets node uses service account credentials for authentication. Ensure permissions are set. | Google Cloud console setup for Sheets API         |
| API response structure assumed as `{ data: { semrushAPI: { trafficSummary: [...] } } }`. Changes in API may require node 3 update. | Maintain API contract awareness                     |