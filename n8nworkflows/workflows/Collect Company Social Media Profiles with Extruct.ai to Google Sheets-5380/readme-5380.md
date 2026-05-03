Collect Company Social Media Profiles with Extruct.ai to Google Sheets

https://n8nworkflows.xyz/workflows/collect-company-social-media-profiles-with-extruct-ai-to-google-sheets-5380


# Collect Company Social Media Profiles with Extruct.ai to Google Sheets

### 1. Workflow Overview

This workflow automates the enrichment of company social media profiles by integrating user input with the Extruct.ai API and storing the enriched data in Google Sheets. The target use case is for users to submit a company name or website via a form, trigger data enrichment through Extruct.ai, poll for completion, and then append or update the results in a Google Sheets document for further analysis or record-keeping.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures user input via a web form and sets necessary parameters.
- **1.2 Data Enrichment and Status Polling:** Sends the input to Extruct.ai for enrichment, then repeatedly checks the processing status until completion.
- **1.3 Data Retrieval and Storage:** Retrieves the enriched data, formats it for Google Sheets, and updates the sheet accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures the company name or website submitted via a form and initializes key variables required for the enrichment process.

**Nodes Involved:**  
- On form submission  
- Variables  
- Sticky Note1

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; listens for form submissions titled "Company Social Presence" with a single field "Name" where users enter the company name or website.  
  - *Configuration:* Webhook-based trigger with form field "Name" and description prompting for company name or website.  
  - *Inputs:* External HTTP POST from form submission.  
  - *Outputs:* Triggers downstream nodes with JSON containing the submitted "Name".  
  - *Edge Cases:* Missing or empty input may cause downstream API calls to fail or return no results.

- **Variables**  
  - *Type:* Set node  
  - *Role:* Holds static workflow parameters like the Extruct.ai table ID.  
  - *Configuration:* Defines a string variable `EXTRUCT_TABLE_ID` where users must input their own Extruct table ID.  
  - *Inputs:* Receives JSON from "On form submission".  
  - *Outputs:* Passes JSON augmented with the `EXTRUCT_TABLE_ID` variable.  
  - *Edge Cases:* If the table ID is missing or incorrect, API calls will fail authentication or data retrieval.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Documentation for the input block, explaining user submission and variable setup.  
  - *Inputs/Outputs:* None.

---

#### 1.2 Data Enrichment and Status Polling

**Overview:**  
This block sends the submitted company data to Extruct.ai for enrichment and then enters a polling loop to repeatedly check the processing status until the enrichment run completes.

**Nodes Involved:**  
- Enrich form input  
- Wait  
- Get status  
- If running  
- Sticky Note2

**Node Details:**

- **Enrich form input**  
  - *Type:* HTTP Request  
  - *Role:* Posts the user-submitted company name to the Extruct.ai API table rows endpoint to initiate enrichment processing.  
  - *Configuration:*  
    - HTTP POST to https://api.extruct.ai/v1/tables/{EXTRUCT_TABLE_ID}/rows  
    - JSON body includes the "input" field populated dynamically from the form submission (`$('On form submission').item.json.Name`).  
    - Uses Bearer token authentication via a stored credential.  
  - *Inputs:* Receives `EXTRUCT_TABLE_ID` and user input from Variables and On form submission nodes.  
  - *Outputs:* Response from API indicating job submission success or failure.  
  - *Edge Cases:*  
    - Unauthorized if token is invalid.  
    - API errors if input is malformed or table ID incorrect.  
    - Network timeouts.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses workflow execution for 10 seconds before checking status again to avoid excessive API calls.  
  - *Configuration:* Fixed 10-second delay.  
  - *Inputs:* Triggered after Enrich form input and again after If running node if enrichment is still in progress.  
  - *Outputs:* Proceeds to Get status node.  
  - *Edge Cases:* Long-running jobs may cause many iterations; workflow run timeouts possible.

- **Get status**  
  - *Type:* HTTP Request  
  - *Role:* Requests the current status of the enrichment job from Extruct.ai by querying the table endpoint.  
  - *Configuration:*  
    - HTTP GET to https://api.extruct.ai/v1/tables/{EXTRUCT_TABLE_ID}  
    - Bearer token authentication.  
  - *Inputs:* Triggered after Wait node.  
  - *Outputs:* JSON response containing `run_status` field.  
  - *Edge Cases:* Authentication failure, API rate limits, or unexpected status formats.

- **If running**  
  - *Type:* If node  
  - *Role:* Conditional check to determine if the enrichment job status is "running".  
  - *Configuration:* Compares `run_status` field from Get status node's JSON to string "running".  
  - *Inputs:* Receives status JSON from Get status.  
  - *Outputs:*  
    - If true: loops back to Wait node to delay before next status check.  
    - If false: proceeds to Get data node to fetch enrichment results.  
  - *Edge Cases:* If API status field is missing or corrupted, condition may fail unexpectedly.

- **Sticky Note2**  
  - *Type:* Sticky Note  
  - *Role:* Documents this enrichment and polling logic, clarifying the looping mechanism until data is ready.

---

#### 1.3 Data Retrieval and Storage

**Overview:**  
Once enrichment is complete, this block fetches the enriched data from Extruct.ai, formats it into a flat structure suitable for Google Sheets, and appends or updates the records in a specified Google Sheets document.

**Nodes Involved:**  
- Get data  
- Format last input  
- Import to Sheets  
- Sticky Note3

**Node Details:**

- **Get data**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the enriched data rows from the Extruct.ai table’s data endpoint.  
  - *Configuration:*  
    - HTTP GET to https://api.extruct.ai/v1/tables/{EXTRUCT_TABLE_ID}/data  
    - Bearer token authentication.  
  - *Inputs:* Triggered from If running node when enrichment status is no longer "running".  
  - *Outputs:* Returns JSON with rows of enriched company profile data.  
  - *Edge Cases:* API errors, empty datasets, authentication issues.

- **Format last input**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes the raw API response to extract the last row of data and flatten nested objects into key-value pairs for Google Sheets.  
  - *Configuration:* Custom JavaScript that:  
    - Extracts the last row from the `rows` array.  
    - Iterates over each data field, extracting `.value.answer` or `.value` fields.  
    - Handles missing or undefined values by inserting empty strings.  
  - *Inputs:* Receives JSON rows from Get data node.  
  - *Outputs:* Single flattened JSON object representing one enriched company record.  
  - *Edge Cases:* If data structure changes or is empty, may produce incomplete output or throw runtime errors.

- **Import to Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Appends or updates the enriched company data in a Google Sheets document.  
  - *Configuration:*  
    - Operation: appendOrUpdate  
    - Requires user to specify the target spreadsheet by URL and the sheet name.  
    - Column mapping must be configured by dragging input fields to sheet columns after a first test run.  
    - Credential: Google Sheets OAuth2 account connected in n8n.  
  - *Inputs:* Receives formatted JSON from Format last input node.  
  - *Outputs:* Updates the Google Sheets document with new or updated rows.  
  - *Edge Cases:*  
    - Incorrect or missing spreadsheet ID or sheet name causes failures.  
    - Google API quota or permission errors.  
    - Improper column mapping results in misaligned data.

- **Sticky Note3**  
  - *Type:* Sticky Note  
  - *Role:* Explains the output process of retrieving, formatting, and saving data to Google Sheets.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                     | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                     |
|---------------------|-------------------|-----------------------------------|------------------------|-----------------------|------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger      | Entry point; capture user input   | —                      | Variables             | ## Input: - User submits a company via the form. - Variables node sets up required parameters (Table ID). |
| Variables           | Set               | Define static variables (Table ID)| On form submission     | Enrich form input     | ## Input: - User submits a company via the form. - Variables node sets up required parameters (Table ID). |
| Enrich form input   | HTTP Request      | Submit input to Extruct.ai for enrichment | Variables              | Wait                  | ## Enrichment: The flow sends the company data for enrichment and waits until processing is complete. It checks the status in a loop until the data is ready. |
| Wait                | Wait              | Delay between status checks       | Enrich form input, If running | Get status         | ## Enrichment: The flow sends the company data for enrichment and waits until processing is complete. It checks the status in a loop until the data is ready. |
| Get status          | HTTP Request      | Check enrichment job status       | Wait                   | If running             | ## Enrichment: The flow sends the company data for enrichment and waits until processing is complete. It checks the status in a loop until the data is ready. |
| If running          | If                | Condition: is enrichment still running? | Get status             | Wait (if running), Get data (if not running) | ## Enrichment: The flow sends the company data for enrichment and waits until processing is complete. It checks the status in a loop until the data is ready. |
| Get data            | HTTP Request      | Retrieve enriched data            | If running (false path) | Format last input      | ## Output: Once enrichment is finished, the flow retrieves the results, formats them, and updates your Google Sheet. |
| Format last input   | Code              | Flatten and prepare data for Sheets | Get data               | Import to Sheets       | ## Output: Once enrichment is finished, the flow retrieves the results, formats them, and updates your Google Sheet. |
| Import to Sheets    | Google Sheets     | Append or update data in Google Sheets | Format last input      | —                      | ## Output: Once enrichment is finished, the flow retrieves the results, formats them, and updates your Google Sheet. |
| Sticky Note         | Sticky Note       | Workflow Quickstart Guide         | —                      | —                      | See detailed Quickstart Guide in node content.                                                  |
| Sticky Note1        | Sticky Note       | Input block documentation         | —                      | —                      | ## Input: - User submits a company via the form. - Variables node sets up required parameters (Table ID). |
| Sticky Note2        | Sticky Note       | Enrichment and polling explanation| —                      | —                      | ## Enrichment: The flow sends the company data for enrichment and waits until processing is complete. It checks the status in a loop until the data is ready. |
| Sticky Note3        | Sticky Note       | Output block documentation        | —                      | —                      | ## Output: Once enrichment is finished, the flow retrieves the results, formats them, and updates your Google Sheet. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "On form submission" node:**  
   - Type: Form Trigger  
   - Configure the form:  
     - Title: "Company Social Presence"  
     - Form fields: One field labeled "Name" (text input)  
     - Description: "Enter the name or website of a company"  
   - Save and note the webhook URL for form submissions.

2. **Create the "Variables" node:**  
   - Type: Set  
   - Add a string variable named `EXTRUCT_TABLE_ID`  
   - Set its value to your Extruct table ID (retrieve from your Extruct web app URL).  
   - Connect "On form submission" → "Variables".

3. **Create the "Enrich form input" node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.extruct.ai/v1/tables/{{ $json.EXTRUCT_TABLE_ID }}/rows`  
   - Authentication: HTTP Bearer Token (using your Extruct API token credential)  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "rows": [
         {
           "data": {
             "input": "{{ $('On form submission').item.json.Name }}"
           }
         }
       ],
       "run": true
     }
     ```  
   - Connect "Variables" → "Enrich form input".

4. **Create the "Wait" node:**  
   - Type: Wait  
   - Set duration to 10 seconds.  
   - Connect "Enrich form input" → "Wait".

5. **Create the "Get status" node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.extruct.ai/v1/tables/{{ $json.EXTRUCT_TABLE_ID }}`  
   - Authentication: HTTP Bearer Token (same credential as above)  
   - Enable "Always Output Data".  
   - Connect "Wait" → "Get status".

6. **Create the "If running" node:**  
   - Type: If  
   - Condition: Check if `status.run_status` equals "running" (case-sensitive, strict validation).  
   - Connect "Get status" → "If running".

7. **Looping connections:**  
   - If **true** (job is running): Connect "If running" → "Wait" (to repeat polling).  
   - If **false** (job finished): Connect "If running" → "Get data".

8. **Create the "Get data" node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.extruct.ai/v1/tables/{{ $json.EXTRUCT_TABLE_ID }}/data`  
   - Authentication: HTTP Bearer Token  
   - Enable "Always Output Data".  
   - Connect "If running" false output → "Get data".

9. **Create the "Format last input" node:**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the following code:  
     ```javascript
     const rows = items[0].json.rows || items[0].json[0]?.rows;
     const lastRow = rows[rows.length - 1];
     const data = lastRow.data;
     let flatRow = {};
     for (const key in data) {
       if (data[key] && data[key].value && data[key].value.answer !== undefined) {
         flatRow[key] = data[key].value.answer;
       } else if (data[key] && data[key].value !== undefined) {
         flatRow[key] = data[key].value;
       } else {
         flatRow[key] = '';
       }
     }
     return [{ json: flatRow }];
     ```  
   - Connect "Get data" → "Format last input".

10. **Create the "Import to Sheets" node:**  
    - Type: Google Sheets  
    - Operation: Append or update  
    - Set the Document ID by pasting the URL of your Google Sheets copy of the provided template.  
    - Specify the Sheet Name where data will be appended/updated.  
    - Authenticate with your Google Sheets OAuth2 credentials in n8n.  
    - After initial test run, map input fields to sheet columns, ensuring "Company name" is set as the key column for updates.  
    - Connect "Format last input" → "Import to Sheets".

11. **Add documentation nodes (Sticky Notes):**  
    - Add sticky notes to document the workflow steps and clarify usage, including the Quickstart Guide, input, enrichment, and output blocks. This aids maintainability and user understanding.

12. **Activate the workflow:**  
    - Switch the workflow to active in n8n.  
    - Use the form URL to submit data and test the entire enrichment and data import process.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                          | Context or Link                                                                                                                                                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Quickstart Guide includes step-by-step setup instructions, API token configuration, and Google Sheets mapping details.                                                                                                                                               | See Sticky Note node content within the workflow for detailed instructions.                                                                                                                                                                     |
| Extruct offers 2,500 free credits for new users signing up at www.extruct.ai/.                                                                                                                                                                                        | https://www.extruct.ai/                                                                                                                                                                                                                         |
| Extruct table template to create your own table and retrieve the table ID: https://app.extruct.ai/tables/shared/VfGemEC1BujAIJx8                                                                                                                                      | https://app.extruct.ai/tables/shared/VfGemEC1BujAIJx8                                                                                                                                                                                          |
| Google Sheets template to use as the destination for enriched data: https://docs.google.com/spreadsheets/d/1eeQ7VIbZ94z2ji9NqRT0bXS8bNtixWbUDopOQeM005w/edit?usp=sharing                                                                                                 | https://docs.google.com/spreadsheets/d/1eeQ7VIbZ94z2ji9NqRT0bXS8bNtixWbUDopOQeM005w/edit?usp=sharing                                                                                                                                           |
| For questions or support, Extruct offers user chat support accessible via their website.                                                                                                                                                                               | Reach out through Extruct chat on www.extruct.ai                                                                                                                                                                                                |
| To extend functionality, add custom columns in both the Extruct table and Google Sheets, and map them accordingly in the workflow.                                                                                                                                   | Documented in the Quickstart Guide Sticky Note.                                                                                                                                                                                                 |

---

**Disclaimer:** The content above is derived exclusively from an n8n automated workflow. It complies fully with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.