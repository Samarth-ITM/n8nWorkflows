Track Website Traffic Data with SEMrush API and Google Sheets

https://n8nworkflows.xyz/workflows/track-website-traffic-data-with-semrush-api-and-google-sheets-7685


# Track Website Traffic Data with SEMrush API and Google Sheets

### 1. Workflow Overview

This workflow automates the process of tracking website traffic data by integrating SEMrush API with Google Sheets via n8n. It is designed for users who want to submit a website URL via a web form, retrieve detailed traffic statistics from SEMrush, and automatically log the summarized data into a Google Sheets document for easy tracking and analysis.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Captures website URLs submitted by users through a web form.
- **1.2 API Data Retrieval:** Sends the submitted website URL to the SEMrush API to fetch traffic data.
- **1.3 Data Processing:** Extracts and reformats the relevant traffic summary data from the API response.
- **1.4 Data Logging:** Appends the processed traffic data as new rows into a Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block waits for users to submit a website URL through a web form interface, triggering the workflow and collecting user input data for subsequent API querying.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - **Type:** Form Trigger  
    - **Technical Role:** Listens for form submissions via a webhook, initiating the workflow.  
    - **Configuration:**  
      - Form Title: "Website Traffic Checker"  
      - One required field: `website` (user must input a website URL)  
      - No additional options configured  
    - **Expressions/Variables:** Outputs the submitted JSON containing the `website` field for downstream nodes.  
    - **Input Connections:** None (entry point)  
    - **Output Connections:** Connects to the "website traffic checker" HTTP Request node  
    - **Version-specific Requirements:** Uses n8n webhook and form trigger version 2.2  
    - **Edge Cases / Potential Failures:**  
      - Missing required field (workflow does not start)  
      - Malformed URLs submitted (could lead to invalid API calls downstream)  
    - **Sub-workflow:** None

#### 1.2 API Data Retrieval

- **Overview:**  
  Sends a POST request with the submitted website URL to the SEMrush API endpoint hosted on RapidAPI, including necessary authentication headers. Retrieves raw traffic data in response.

- **Nodes Involved:**  
  - website traffic checker

- **Node Details:**

  - **website traffic checker**  
    - **Type:** HTTP Request  
    - **Technical Role:** Performs a POST request to the SEMrush API to obtain website traffic metrics.  
    - **Configuration:**  
      - URL: `https://semrush-website-traffic-checker.p.rapidapi.com/webtraffic.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameters: includes one parameter `website` dynamically set from the form input `{{$json.website}}`  
      - Header Parameters:  
        - `x-rapidapi-host`: semrush-website-traffic-checker.p.rapidapi.com  
        - `x-rapidapi-key`: user must provide their RapidAPI key (placeholder `"your key"` in config)  
      - Sends both headers and body  
    - **Expressions/Variables:** Uses expression to inject website URL dynamically from form submission  
    - **Input Connections:** From "On form submission" node  
    - **Output Connections:** To "Reformat" node  
    - **Version-specific Requirements:** HTTP Request node version 4.2 or higher recommended for multipart-form-data support  
    - **Edge Cases / Potential Failures:**  
      - Invalid or missing API key results in authentication failure  
      - API rate limits or downtime cause request failure  
      - Invalid website URL may cause API to return errors or empty data  
      - Network issues causing timeouts or failed requests  
    - **Sub-workflow:** None

#### 1.3 Data Processing

- **Overview:**  
  Extracts the relevant `trafficSummary` object from the raw SEMrush API response, simplifying the dataset for easier insertion into Google Sheets.

- **Nodes Involved:**  
  - Reformat

- **Node Details:**

  - **Reformat**  
    - **Type:** Code (JavaScript)  
    - **Technical Role:** Processes the raw JSON response by extracting the nested `trafficSummary` data field.  
    - **Configuration:**  
      - Script: `return $input.first().json.data.semrushAPI.trafficSummary;`  
      - This returns the `trafficSummary` portion of the API response JSON for downstream use.  
    - **Expressions/Variables:** Accesses nested JSON properties to isolate traffic summary  
    - **Input Connections:** From "website traffic checker" node  
    - **Output Connections:** To "Append Data In Google Sheets" node  
    - **Version-specific Requirements:** Requires n8n Code node version 2 or higher  
    - **Edge Cases / Potential Failures:**  
      - If the API response structure changes and `trafficSummary` is missing, this node will fail or return undefined  
      - If input JSON is malformed or empty, script execution errors may occur  
    - **Sub-workflow:** None

#### 1.4 Data Logging

- **Overview:**  
  Appends the extracted traffic summary data into a designated Google Sheets document row, enabling ongoing logging and tracking of website traffic metrics.

- **Nodes Involved:**  
  - Append Data In Google Sheets

- **Node Details:**

  - **Append Data In Google Sheets**  
    - **Type:** Google Sheets node  
    - **Technical Role:** Appends one or more rows of data into a specified Google Sheets spreadsheet and sheet.  
    - **Configuration:**  
      - Operation: append  
      - Document ID and Sheet Name: dynamically configured (value hidden in JSON)  
      - Authentication: serviceAccount credentials (using a Google Docs service account)  
    - **Expressions/Variables:** Accepts input JSON containing the traffic summary fields to map into columns  
    - **Input Connections:** From "Reformat" node  
    - **Output Connections:** None (end node)  
    - **Version-specific Requirements:** Node version 4.6+ recommended for stable Google Sheets API integration  
    - **Edge Cases / Potential Failures:**  
      - Invalid or expired Google API credentials cause authentication failure  
      - Incorrect document ID or sheet name results in write errors  
      - Google Sheets API rate limits or quota exceeded  
      - Data format mismatches may cause append failures  
    - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                                      | Input Node(s)          | Output Node(s)                    | Sticky Note                                                                                               |
|---------------------------|-------------------|-----------------------------------------------------|-----------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------|
| On form submission        | Form Trigger      | Triggers workflow on website URL form submission    | None                  | website traffic checker           | Triggers the workflow whenever a user submits the website URL through the form. It collects input data.  |
| website traffic checker   | HTTP Request      | Sends website to SEMrush API and fetches traffic data | On form submission    | Reformat                         | Sends a POST request to SEMrush API with the submitted website and includes RapidAPI key for authentication. |
| Reformat                 | Code (JavaScript) | Extracts `trafficSummary` from API response          | website traffic checker | Append Data In Google Sheets      | Extracts the `trafficSummary` object, simplifying data for subsequent use.                                |
| Append Data In Google Sheets | Google Sheets     | Appends traffic data as new row in Google Sheets    | Reformat               | None                             | Appends formatted traffic data as a new row into specified Google Sheets for automatic logging.          |
| Sticky Note              | Sticky Note       | Workflow description and summary                      | None                  | None                             | **Automated Website Traffic Checker with Google Sheets Integration** ... (full content in section 1.1)   |
| Sticky Note1             | Sticky Note       | Notes on On form submission node                      | None                  | None                             | On form submission node triggers the workflow on form input.                                             |
| Sticky Note2             | Sticky Note       | Notes on website traffic checker node                 | None                  | None                             | Sends POST request to SEMrush API with authentication headers.                                            |
| Sticky Note3             | Sticky Note       | Notes on Reformat node                                | None                  | None                             | Extracts `trafficSummary` from API response for simpler data handling.                                   |
| Sticky Note4             | Sticky Note       | Notes on Append Data In Google Sheets node            | None                  | None                             | Appends traffic data into Google Sheets for tracking.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "On form submission" node:**
   - Type: Form Trigger  
   - Parameters:  
     - Form Title: "Website Traffic Checker"  
     - Form Description: "Website Traffic Checker"  
     - Form Fields: Add one field  
       - Label: `website`  
       - Type: Text (default)  
       - Required: Yes  
   - This node acts as the webhook entry point triggered by user submissions.

2. **Create the "website traffic checker" node:**
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://semrush-website-traffic-checker.p.rapidapi.com/webtraffic.php`  
     - Method: POST  
     - Content-Type: multipart/form-data  
     - Send Headers: Enabled  
     - Send Body: Enabled  
     - Body Parameters: Add parameter  
       - Name: `website`  
       - Value Expression: `{{$json["website"]}}` (dynamic from form submission)  
     - Header Parameters:  
       - `x-rapidapi-host`: `semrush-website-traffic-checker.p.rapidapi.com`  
       - `x-rapidapi-key`: `<Your_RapidAPI_Key>` (replace with your actual API key)  
   - Connect output of "On form submission" node to input of this node.

3. **Create the "Reformat" node:**
   - Type: Code (JavaScript)  
   - Parameters:  
     - Code:  
       ```javascript
       return $input.first().json.data.semrushAPI.trafficSummary;
       ```  
   - Connect output of "website traffic checker" node to input of this node.

4. **Create the "Append Data In Google Sheets" node:**
   - Type: Google Sheets  
   - Parameters:  
     - Operation: Append  
     - Document ID: Set the Google Sheet document ID where data will be appended  
     - Sheet Name: Set the name of the sheet/tab within the document  
     - Authentication: Service Account  
   - Credentials: Configure a Google API Service Account credential with appropriate Sheets API permissions  
   - Connect output of "Reformat" node to input of this node.

5. **Test the workflow:**
   - Publish the form webhook URL generated by "On form submission" node.  
   - Submit a test website URL via the form.  
   - Verify traffic data is correctly fetched, reformatted, and appended as a new row in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow leverages the SEMrush Website Traffic Checker API available via RapidAPI, requiring a valid RapidAPI subscription and API key for authentication. Ensure your key is kept secure and has appropriate quota.    | https://rapidapi.com/semrush/api/website-traffic-checker                                                    |
| Google Sheets node uses Service Account authentication; create a service account in Google Cloud Console with Sheets API enabled and share the target sheet with the service account email.                                  | https://developers.google.com/identity/protocols/oauth2/service-account                                       |
| For detailed n8n form trigger usage and webhook setup, consult official documentation to properly deploy and expose the form URL.                                                                                           | https://docs.n8n.io/nodes/n8n-nodes-base.formTrigger/                                                        |
| The workflow can be extended to handle error responses gracefully by adding error handling nodes after the HTTP Request or Code nodes, e.g., to notify via email or retry requests on failure.                                 | General best practice in n8n for production workflows                                                        |

---

**Disclaimer:** The content provided is extracted exclusively from an automated workflow created with n8n, respecting all content policies. No illegal, offensive, or protected data is included. All data processed is legal and public.