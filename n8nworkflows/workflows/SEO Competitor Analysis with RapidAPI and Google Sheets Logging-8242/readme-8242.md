SEO Competitor Analysis with RapidAPI and Google Sheets Logging

https://n8nworkflows.xyz/workflows/seo-competitor-analysis-with-rapidapi-and-google-sheets-logging-8242


# SEO Competitor Analysis with RapidAPI and Google Sheets Logging

### 1. Workflow Overview

This workflow automates SEO competitor analysis by accepting a website domain via a user-submitted form, querying a competitor analysis API through RapidAPI, and logging the results into Google Sheets. It is designed for SEO professionals, marketing teams, or agencies who want to efficiently gather and record competitor insights for organic search performance.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures website domain input from a user-submitted form.
- **1.2 Data Preparation and API Request:** Stores input data and sends a POST request to the RapidAPI competitor analysis endpoint.
- **1.3 Response Validation:** Checks if the API response contains valid data.
- **1.4 Data Logging:** Depending on the validation, either logs the API response data or logs a "Not Found" entry into Google Sheets.
- **1.5 Workflow Control:** Introduces a wait period to pace execution and avoid API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Initiates the workflow by capturing a website domain submitted by a user through an embedded form.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: `formTrigger`  
    - Role: Triggers workflow when a user submits the form with a required field `Website`.  
    - Configuration: Form titled "Competitor Analysis" with one required field for domain input (e.g., "instagram.com").  
    - Inputs: None (trigger node)  
    - Outputs: Triggers downstream nodes with form submission data.  
    - Edge Cases: Form submission without a website is prevented by required field setting.

#### 1.2 Data Preparation and API Request

- **Overview:**  
  Stores the submitted website in a global variable for reuse and sends it as a POST request to the RapidAPI competitor analysis endpoint to retrieve competitor data.

- **Nodes Involved:**  
  - Global Storage  
  - Competitor Analysis Request

- **Node Details:**  
  - **Global Storage**  
    - Type: `set`  
    - Role: Stores the submitted website domain into a variable named `website` for reference in later nodes.  
    - Configuration: Assigns `website` from the incoming JSON property `$json.Website`.  
    - Inputs: Output of "On form submission" node  
    - Outputs: Passes stored website data downstream.  
    - Edge Cases: Empty or malformed website input could cause downstream API errors. No explicit validation beyond form required field.  

  - **Competitor Analysis Request**  
    - Type: `httpRequest`  
    - Role: Sends a POST request to RapidAPI endpoint `competitor.php` with the stored website domain to fetch SEO competitor data.  
    - Configuration:  
      - URL: `https://competitor-analysis2.p.rapidapi.com/competitor.php`  
      - Method: POST  
      - Content-Type: multipart-form-data  
      - Headers: Required RapidAPI host and API key (`x-rapidapi-host` and `x-rapidapi-key`)  
      - Body: Sends website parameter with value from `={{ $json.website }}`  
    - Inputs: Output from "Global Storage"  
    - Outputs: API response JSON with competitor data  
    - Edge Cases:  
      - Network errors, authentication failures with RapidAPI key  
      - Malformed or missing API response data  
      - API rate limiting  
    - Version notes: Requires valid RapidAPI key and proper header configuration.

#### 1.3 Response Validation

- **Overview:**  
  Verifies if the API response contains valid and non-empty competitor data before proceeding.

- **Nodes Involved:**  
  - If

- **Node Details:**  
  - **If**  
    - Type: `if`  
    - Role: Checks that the API response's `data` object and its nested arrays `organicCompetitors`, `organicPages`, and `domainOrganicSearchKeywords` are non-empty.  
    - Configuration: Conditions use strict type validation to confirm:  
      - `data` object is not empty  
      - `data.semrushAPI.organicCompetitors` array is not empty  
      - `data.semrushAPI.organicPages` array is not empty  
      - `data.semrushAPI.domainOrganicSearchKeywords` array is not empty  
    - Inputs: Output from "Competitor Analysis Request"  
    - Outputs:  
      - True branch if all conditions met (valid data)  
      - False branch if any condition fails (missing or empty data)  
    - Edge Cases:  
      - Partial or malformed API responses causing condition failures  
      - Expression evaluation errors if expected JSON paths are missing.

#### 1.4 Data Logging

- **Overview:**  
  Appends competitor analysis data into Google Sheets if data is valid; otherwise, logs a default "Not Found" record. This maintains a consistent data log in case of failures or missing info.

- **Nodes Involved:**  
  - Google Sheets (true branch)  
  - Wait (false branch)  
  - Google Sheets1 (after wait node)

- **Node Details:**  
  - **Google Sheets**  
    - Type: `googleSheets`  
    - Role: Appends rows with competitor data extracted from the API response into a specified Google Sheet.  
    - Configuration:  
      - Operation: Append  
      - Sheet Name: Sheet1 (gid=0)  
      - Document ID: Configured with a Google Sheets document ID (assumed pre-configured)  
      - Authentication: Service account credentials  
      - Columns mapped as:  
        - Domain: `={{ $('Global Storage').item.json.website }}`  
        - Organic Page: `={{ $json.data.semrushAPI.organicPages }}`  
        - Organic competitor: `={{ $json.data.semrushAPI.organicCompetitors }}`  
        - Domain organic search: `={{ $json.data.semrushAPI.domainOrganicSearchKeywords }}`  
    - Inputs: True branch of "If" node  
    - Outputs: Ends workflow here on success  
    - Edge Cases:  
      - Google API auth errors  
      - Data mapping errors if response structure changes  
      - Large payloads may require pagination or batching.

  - **Wait**  
    - Type: `wait`  
    - Role: Introduces a delay (default 5 seconds) before proceeding to log missing data, controlling workflow pacing and API rate limits.  
    - Configuration: Default wait time with a webhook ID for pause control  
    - Inputs: False branch of "If" node  
    - Outputs: Connects to "Google Sheets1" node  
    - Edge Cases: Workflow delay may cause timeouts if downstream systems expect quicker responses.

  - **Google Sheets1**  
    - Type: `googleSheets`  
    - Role: Appends a single record marking "Not data found." for organic pages, competitors, and search keywords when the API returns no valid data.  
    - Configuration:  
      - Operation: Append  
      - Sheet Name: Sheet1 (gid=0)  
      - Document ID: Same Google Sheets document as above  
      - Authentication: Service account credentials  
      - Columns mapped as:  
        - Domain: `={{ $('Global Storage').item.json.website }}`  
        - Organic Page: `"Not data found."`  
        - Organic competitor: `"Not data found."`  
        - Domain organic search: `"Not data found."`  
    - Inputs: Output of "Wait" node  
    - Outputs: Workflow ends here after logging missing data  
    - Edge Cases: Same Google Sheets API considerations as above.

#### 1.5 Workflow Control and Metadata

- **Sticky Notes:**  
  Provide detailed descriptions of each block and node purpose for easier workflow understanding and maintenance.

- **Metadata:**  
  - Workflow ID and instance data (internal to n8n)  
  - Credential references for Google Sheets and RapidAPI integration (must be configured prior to execution).

---

### 3. Summary Table

| Node Name                 | Node Type       | Functional Role                                  | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                          |
|---------------------------|-----------------|-------------------------------------------------|--------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| On form submission        | formTrigger     | Captures website domain from user form input    | None                     | Global Storage           | ### 1. 🟢 **On form submission** Trigger when user submits form with Website field                  |
| Global Storage            | set             | Stores website domain for downstream use        | On form submission       | Competitor Analysis Request | ### 2. 📦 **Global Storage** Stores website value from form for later use                           |
| Competitor Analysis Request | httpRequest    | Sends POST request to RapidAPI competitor API   | Global Storage           | If                       | ### 3. 🌐 **Competitor Analysis Request** Sends competitor analysis API request                     |
| If                        | if              | Checks if API response contains valid data      | Competitor Analysis Request | Google Sheets (true), Wait (false) | ### 4. ⚖️ **Condition Checking** Validates API response data presence                              |
| Google Sheets             | googleSheets    | Logs competitor data into Google Sheets          | If (true)                | None                     | ### 5. 📊 **Google Sheets - Insert Record from Response Body** Maps API response data to sheet     |
| Wait                      | wait            | Waits 5 seconds before proceeding                 | If (false)               | Google Sheets1           | ### 6. ⏳ **Wait Node (5-Second Interval)** Controls workflow pacing                               |
| Google Sheets1            | googleSheets    | Logs "Not Found" entry when no data available    | Wait                     | None                     | ### 7. 📊 **Google Sheets - Insert 'Not Found' Record** Logs missing data notification              |
| Sticky Note1              | stickyNote      | Documentation node for On form submission        | None                     | None                     |                                                                                                    |
| Sticky Note2              | stickyNote      | Documentation node for Global Storage             | None                     | None                     |                                                                                                    |
| Sticky Note3              | stickyNote      | Documentation node for Competitor Analysis Request | None                   | None                     |                                                                                                    |
| Sticky Note               | stickyNote      | Documentation node for If condition               | None                     | None                     |                                                                                                    |
| Sticky Note4              | stickyNote      | Documentation node for Wait node                   | None                     | None                     |                                                                                                    |
| Sticky Note5              | stickyNote      | Documentation node for Google Sheets data insertion | None                   | None                     |                                                                                                    |
| Sticky Note6              | stickyNote      | Documentation node for Google Sheets "Not Found" logging | None                 | None                     |                                                                                                    |
| Sticky Note7              | stickyNote      | Overview and description of entire workflow       | None                     | None                     | # Automated SEO Competitor Analysis and Google Sheets Logging (see detailed description in node)  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**
   - Node Type: `formTrigger`  
   - Name: "On form submission"  
   - Configuration:  
     - Form Title: "Competitor Analysis"  
     - Form Fields:  
       - Field Label: "Website"  
       - Placeholder: "e.g. instagram.com"  
       - Required: Yes  
   - This node triggers the workflow upon user submission.

2. **Create a Set Node for Global Storage**
   - Node Type: `set`  
   - Name: "Global Storage"  
   - Configuration:  
     - Add a field named `website`  
     - Value: Expression `{{$json.Website}}` (captures form input)  
   - Connect "On form submission" output to this node.

3. **Create an HTTP Request Node for Competitor Analysis**
   - Node Type: `httpRequest`  
   - Name: "Competitor Analysis Request"  
   - Configuration:  
     - URL: `https://competitor-analysis2.p.rapidapi.com/competitor.php`  
     - HTTP Method: POST  
     - Content Type: multipart-form-data  
     - Headers:  
       - `x-rapidapi-host`: `competitor-analysis2.p.rapidapi.com`  
       - `x-rapidapi-key`: Your RapidAPI key (must be set before running)  
     - Body Parameters:  
       - Name: `website`  
       - Value: Expression `{{$json.website}}` (from Global Storage node)  
   - Connect "Global Storage" output to this node.

4. **Create an If Node for Response Validation**
   - Node Type: `if`  
   - Name: "If"  
   - Configuration: Use version 2 of conditions with strict type validation.  
     - Conditions: All must be true (AND combinator):  
       1. `$json.data` is not empty (operation: object notEmpty)  
       2. `$json.data.semrushAPI.organicCompetitors` array is not empty (operation: array notEmpty)  
       3. `$json.data.semrushAPI.organicPages` array is not empty (operation: array notEmpty)  
       4. `$json.data.semrushAPI.domainOrganicSearchKeywords` array is not empty (operation: array notEmpty)  
   - Connect "Competitor Analysis Request" output to this node.

5. **Create Google Sheets Node to Log Valid Data**
   - Node Type: `googleSheets`  
   - Name: "Google Sheets"  
   - Configuration:  
     - Operation: Append  
     - Document ID: Set your Google Sheets document ID  
     - Sheet Name: "Sheet1" (or your target sheet)  
     - Authentication: Use Service Account credentials configured in n8n  
     - Columns Mapping:  
       - Domain: Expression `={{ $('Global Storage').item.json.website }}`  
       - Organic Page: Expression `={{ $json.data.semrushAPI.organicPages }}`  
       - Organic competitor: Expression `={{ $json.data.semrushAPI.organicCompetitors }}`  
       - Domain organic search: Expression `={{ $json.data.semrushAPI.domainOrganicSearchKeywords }}`  
   - Connect "If" node **true** output to this node.

6. **Create Wait Node for Delay**
   - Node Type: `wait`  
   - Name: "Wait"  
   - Configuration: Use default wait time (5 seconds) or set explicitly via parameters.  
   - Connect "If" node **false** output to this node.

7. **Create Google Sheets Node to Log "Not Found" Data**
   - Node Type: `googleSheets`  
   - Name: "Google Sheets1"  
   - Configuration:  
     - Operation: Append  
     - Document ID: Use the same as previous Google Sheets node  
     - Sheet Name: "Sheet1"  
     - Authentication: Use same Google Sheets service account  
     - Columns Mapping:  
       - Domain: Expression `={{ $('Global Storage').item.json.website }}`  
       - Organic Page: Static text `"Not data found."`  
       - Organic competitor: Static text `"Not data found."`  
       - Domain organic search: Static text `"Not data found."`  
   - Connect output of "Wait" node to this node.

8. **Connect Workflow Ends**
   - The "Google Sheets" node ends the workflow on success.  
   - The "Google Sheets1" node ends the workflow for missing data.

9. **Set Credentials**
   - Configure Google Sheets credentials with a service account that has write access to your spreadsheet.  
   - Configure RapidAPI credentials by setting your valid API key in the HTTP Request node headers.

10. **Optional: Add Sticky Notes**
    - Create sticky notes for documentation near each node or block for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                     | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Ensure to configure your RapidAPI key and Google Sheets credentials before running this workflow to avoid authentication errors.                                                                                                                                 | Workflow credentials setup                                                                              |
| The workflow uses a 5-second wait node to help respect API rate limits and prevent throttling. Adjust this delay based on your API plan restrictions.                                                                                                            | Rate limiting and API pacing                                                                            |
| The workflow expects the API response JSON to include `semrushAPI` object with arrays: `organicCompetitors`, `organicPages`, and `domainOrganicSearchKeywords`. Changes in API response structure may require workflow adjustments.                                | API response structure dependencies                                                                    |
| This workflow is suitable for SEO professionals and agencies who want automated competitor insights and historical logging in Google Sheets.                                                                                                                   | Use case and target audience                                                                             |
| For more information on n8n formTrigger nodes, see: https://docs.n8n.io/nodes/n8n-nodes-base.formTrigger/                                                                                                                                                       | Official n8n documentation                                                                              |
| For Google Sheets integration details: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleSheets/                                                                                                                                          | Official n8n Google Sheets node documentation                                                          |
| RapidAPI competitor analysis endpoint documentation is required to understand API parameters and response format. Ensure to read official RapidAPI docs for `competitor-analysis2` service.                                                                      | RapidAPI docs (external)                                                                                 |
| Sticky notes inside the workflow provide detailed block descriptions for maintainers and auditors.                                                                                                                        | Workflow internal documentation                                                                         |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.