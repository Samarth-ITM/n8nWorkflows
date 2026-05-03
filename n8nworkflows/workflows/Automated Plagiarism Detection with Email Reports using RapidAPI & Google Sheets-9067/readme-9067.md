Automated Plagiarism Detection with Email Reports using RapidAPI & Google Sheets

https://n8nworkflows.xyz/workflows/automated-plagiarism-detection-with-email-reports-using-rapidapi---google-sheets-9067


# Automated Plagiarism Detection with Email Reports using RapidAPI & Google Sheets

### 1. Workflow Overview

This workflow automates plagiarism detection for text content submitted via a Google Sheet. Upon detecting new entries, it sends the content to a third-party plagiarism-checking API (via RapidAPI), processes the results, generates a detailed HTML report, and emails it to the user. The workflow also updates the Google Sheet with the status of the check ("Success" or "Failed") and sends an alert email to IT if the check fails.

**Primary Use Cases:**  
- Academic institutions validating student submissions  
- Content teams ensuring originality before publishing  
- Editors and educators automating plagiarism checks  

**Logical Blocks:**  
- **1.1 Input Reception:** Watches Google Sheets for new rows with submitted content.  
- **1.2 Plagiarism API Integration:** Sends content to the plagiarism-checking API and receives response.  
- **1.3 Response Validation:** Verifies API response success and routes accordingly.  
- **1.4 Results Extraction:** Extracts raw plagiarism results from the API response.  
- **1.5 Report Generation:** Formats the results into a readable HTML email report.  
- **1.6 User Notification:** Sends the plagiarism report email to the user.  
- **1.7 Status Logging:** Updates the Google Sheet row status to "Success" or "Failed."  
- **1.8 Failure Handling:** Sends alert emails to IT and marks failure in the sheet if API check fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Monitors a specific Google Sheet for new rows added. On detecting new content, triggers the workflow to start the plagiarism check.

- **Nodes Involved:**  
  - Trigger - New Row in Google Sheet  
  - Sticky Note (explaining the trigger)

- **Node Details:**  
  - **Trigger - New Row in Google Sheet**  
    - Type: Google Sheets Trigger  
    - Role: Detects new rows in the configured sheet and triggers the workflow.  
    - Config: Polls every minute on Sheet1 (gid=0) of the specified Google Sheet document.  
    - Credential: OAuth2 Google Sheets Trigger account for authentication.  
    - Inputs: None (trigger node)  
    - Outputs: New row data including "Content" field  
    - Potential Failures: OAuth token expiry, Google API rate limits, incorrect sheet ID or name.  
    - Sticky Note: Describes monitoring new Google Sheet rows for content submission.

#### 1.2 Plagiarism API Integration

- **Overview:**  
  Sends the submitted content to the external plagiarism API using a POST request with multipart/form-data. Includes required headers for API authentication.

- **Nodes Involved:**  
  - Send Content to Plagiarism API  
  - Sticky Note

- **Node Details:**  
  - **Send Content to Plagiarism API**  
    - Type: HTTP Request  
    - Role: Posts content to RapidAPI plagiarism-checker endpoint.  
    - Config:  
      - URL: `https://plagiarism-checker-ai-powered.p.rapidapi.com/plagiarism.php`  
      - Method: POST  
      - Body: Multipart-form-data with field "content" set from Google Sheet field `Content`.  
      - Headers: `x-rapidapi-host` and `x-rapidapi-key` (API key must be provided)  
    - Inputs: Content from trigger  
    - Outputs: API JSON response  
    - Potential Failures: Network errors, invalid API key, API downtime, invalid content format.  
    - Sticky Note: Notes the sending of content and use of headers.

#### 1.3 Response Validation

- **Overview:**  
  Validates if the API response indicates success and contains non-empty plagiarism results. Routes the flow to success or failure branches accordingly.

- **Nodes Involved:**  
  - Check API Response Success (If Node)  
  - Sticky Note

- **Node Details:**  
  - **Check API Response Success**  
    - Type: If Node  
    - Role: Checks two conditions:  
      1. `$json.success` is boolean true  
      2. `$json.data.results.results` array is not empty  
    - Inputs: API response JSON  
    - Outputs:  
      - True branch: Proceed to result extraction  
      - False branch: Trigger failure handling (email alert and status update)  
    - Potential Failures: Missing or malformed fields, unexpected API schema changes.  
    - Sticky Note: Explains success check and branching logic.

#### 1.4 Results Extraction

- **Overview:**  
  Extracts the nested array of plagiarism results from the API response for further processing.

- **Nodes Involved:**  
  - Extract Plagiarism Results (Code Node)  
  - Sticky Note

- **Node Details:**  
  - **Extract Plagiarism Results**  
    - Type: Code (JavaScript)  
    - Role: Returns the array at `data.results.results` from the first input JSON.  
    - Inputs: API response JSON  
    - Outputs: Array of plagiarism results objects (each with phrase and matched results)  
    - Potential Failures: If the path is missing or data is null, output could be empty or cause errors downstream.  
    - Sticky Note: Notes extraction of raw results.

#### 1.5 Report Generation

- **Overview:**  
  Converts the extracted plagiarism results into a detailed, styled HTML report suitable for email. Includes original phrases, matched sources with links, similarity scores, and matched sentences.

- **Nodes Involved:**  
  - Generate HTML Plagiarism Report (Code Node)  
  - Sticky Note

- **Node Details:**  
  - **Generate HTML Plagiarism Report**  
    - Type: Code (JavaScript)  
    - Role: Iterates over all results to build an HTML string containing:  
      - Report header and introduction  
      - For each phrase: original text, matched sources as clickable links, similarity scores, and example sentences  
      - Marks phrases with no matches clearly  
    - Inputs: Array of plagiarism results from previous node  
    - Outputs: JSON with `subject` and `body` fields for email  
    - Potential Failures: Unexpected data structure, null or missing fields in results, encoding issues in HTML.  
    - Sticky Note: Describes formatting logic and HTML report construction.

#### 1.6 User Notification

- **Overview:**  
  Sends the generated HTML plagiarism report to the user’s email using SMTP.

- **Nodes Involved:**  
  - Send Report to User via Email  
  - Sticky Note

- **Node Details:**  
  - **Send Report to User via Email**  
    - Type: Email Send  
    - Role: Sends email with subject and HTML body from previous node.  
    - Config:  
      - To: user@test.com (placeholder, must be configured per deployment)  
      - From: admin@test.com  
      - Credentials: SMTP account credentials configured in n8n  
    - Inputs: JSON with subject and HTML body  
    - Outputs: Confirmation of email sent  
    - Potential Failures: SMTP authentication failures, invalid email addresses, network issues.  
    - Sticky Note: Notes email sending with SMTP.

#### 1.7 Status Logging

- **Overview:**  
  Updates the original Google Sheet row’s "Status" column to "Success" or "Failed" depending on the check outcome.

- **Nodes Involved:**  
  - Mark Status: Success in Google Sheet  
  - Mark Status: Failed in Google Sheet  
  - Sticky Notes

- **Node Details:**  
  - **Mark Status: Success in Google Sheet**  
    - Type: Google Sheets  
    - Role: Updates the row matching the original content with `"Status":"Success"`  
    - Config: Uses service account authentication, matches on "Content" column, updates Sheet1 (gid=0)  
    - Inputs: Original content from trigger node  
    - Outputs: Confirmation of update  
    - Potential Failures: Row matching errors, permission issues, API limits.  
    - Sticky Note: Notes updating status to Success.

  - **Mark Status: Failed in Google Sheet**  
    - Type: Google Sheets  
    - Role: Similar to above but sets `"Status":"Failed"`  
    - Inputs: Original content from trigger node  
    - Outputs: Confirmation of update  
    - Potential Failures: Same as success node.  
    - Sticky Note: Notes updating status to Failed.

#### 1.8 Failure Handling

- **Overview:**  
  Sends an alert email to the IT team if the plagiarism check fails, including content and troubleshooting instructions.

- **Nodes Involved:**  
  - Send Failure Alert to IT  
  - Sticky Note

- **Node Details:**  
  - **Send Failure Alert to IT**  
    - Type: Email Send  
    - Role: Sends alert email with failure details, user content, timestamp, and recommended actions.  
    - Config:  
      - To: it@test.com  
      - From: admin@test.com  
      - Credentials: SMTP account  
      - HTML body includes dynamic content and timestamp  
    - Inputs: Failure branch from API success check, original content field  
    - Outputs: Confirmation of alert email sent  
    - Potential Failures: SMTP issues, missing content data, email delivery failures.  
    - Sticky Note: Explains alert email contents and purpose.

---

### 3. Summary Table

| Node Name                        | Node Type               | Functional Role                      | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                   |
|---------------------------------|-------------------------|------------------------------------|---------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Trigger - New Row in Google Sheet | Google Sheets Trigger   | Detects new Google Sheet entries   | None                            | Send Content to Plagiarism API  | Watches for a new row added to a specific Google Sheet. Triggers the workflow when new content is submitted.  |
| Send Content to Plagiarism API   | HTTP Request            | Sends content to plagiarism API    | Trigger - New Row in Google Sheet | Check API Response Success      | Sends the submitted content to the external plagiarism-checking API. Uses POST with headers and content body. |
| Check API Response Success       | If                      | Validates API response success     | Send Content to Plagiarism API  | Extract Plagiarism Results (true) / Send Failure Alert to IT (false) | Checks if the API returned success and valid results. Routes flow accordingly.                                |
| Extract Plagiarism Results       | Code                    | Extracts plagiarism results array  | Check API Response Success (true) | Generate HTML Plagiarism Report | Extracts the results array from the API response for formatting.                                             |
| Generate HTML Plagiarism Report  | Code                    | Formats results into HTML report   | Extract Plagiarism Results      | Send Report to User via Email   | Formats plagiarism data into a readable HTML report with links, scores, and matched sentences.                |
| Send Report to User via Email    | Email Send              | Sends plagiarism report email      | Generate HTML Plagiarism Report | Mark Status: Success in Google Sheet | Sends the generated report via SMTP email to the user.                                                       |
| Mark Status: Success in Google Sheet | Google Sheets         | Updates Google Sheet row status to Success | Send Report to User via Email   | None                            | Updates the original row status to "Success".                                                                |
| Send Failure Alert to IT         | Email Send              | Sends failure alert email to IT    | Check API Response Success (false) | Mark Status: Failed in Google Sheet | Sends alert email with failure details and troubleshooting tips.                                            |
| Mark Status: Failed in Google Sheet | Google Sheets         | Updates Google Sheet row status to Failed | Send Failure Alert to IT         | None                            | Marks the row status as "Failed" to track unsuccessful checks.                                               |
| Sticky Note (various)            | Sticky Note             | Provides explanations and comments | Various                        | Various                        | See sticky note content per node as detailed in block analysis sections and summary table.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Node Type: Google Sheets Trigger  
   - Configure to watch a specific Google Sheet document and Sheet1 (gid=0) for new rows.  
   - Poll every minute.  
   - Authenticate with OAuth2 credentials for Sheets.  
   - Output: New row JSON including a "Content" field.

2. **Add HTTP Request Node**  
   - Name: Send Content to Plagiarism API  
   - Type: HTTP Request  
   - URL: `https://plagiarism-checker-ai-powered.p.rapidapi.com/plagiarism.php`  
   - Method: POST  
   - Content-Type: multipart/form-data  
   - Body Parameter: Set "content" to `{{$node["Trigger - New Row in Google Sheet"].json["Content"]}}`  
   - Headers:  
     - `x-rapidapi-host`: `plagiarism-checker-ai-powered.p.rapidapi.com`  
     - `x-rapidapi-key`: *Your RapidAPI key*  
   - Connect trigger node output to this node.

3. **Add If Node**  
   - Name: Check API Response Success  
   - Conditions (AND):  
     - Expression: `{{$json.success}}` is boolean true  
     - Expression: Array at `{{$json.data.results.results}}` is not empty  
   - Connect HTTP Request output to this node.

4. **Add Code Node (Extract Results)**  
   - Name: Extract Plagiarism Results  
   - JavaScript code:  
     ```js
     return $input.first().json.data.results.results;
     ```  
   - Connect If node’s True output to this node.

5. **Add Code Node (Generate HTML Report)**  
   - Name: Generate HTML Plagiarism Report  
   - JavaScript code to iterate over input items and build an HTML report including:  
     - Header and intro paragraph  
     - For each phrase, display the original phrase and matched sources with link, similarity score, and matched sentence  
     - Mark phrases with no matches explicitly  
   - Connect Extract Results node to this node.

6. **Add Email Send Node (User Report)**  
   - Name: Send Report to User via Email  
   - Configure SMTP credentials with valid email account.  
   - To: `user@test.com` (replace with actual user email or dynamic field)  
   - From: `admin@test.com` (replace accordingly)  
   - Subject: `{{$json.subject}}`  
   - HTML Body: `{{$json.body}}`  
   - Connect Generate HTML Report node to this node.

7. **Add Google Sheets Node (Mark Success)**  
   - Name: Mark Status: Success in Google Sheet  
   - Operation: Update row  
   - Sheet: Sheet1 (gid=0) in the same Google Sheet  
   - Matching Column: "Content" (to match the original content)  
   - Update "Status" column to `"Success"`  
   - Authenticate with Google Sheets API via service account.  
   - Connect Email Send (User Report) node output to this node.

8. **Add Email Send Node (Failure Alert)**  
   - Name: Send Failure Alert to IT  
   - Configure SMTP credentials.  
   - To: `it@test.com` (IT team email)  
   - From: `admin@test.com`  
   - Subject: `"ALERT: Plagiarism Check Failed – Immediate Attention Required"`  
   - HTML Body: Include timestamp, user content, failure note, and recommended actions (as per original workflow). Use expressions to insert dynamic data.  
   - Connect If node’s False output to this node.

9. **Add Google Sheets Node (Mark Failed)**  
   - Name: Mark Status: Failed in Google Sheet  
   - Same config as Success node but update "Status" column to `"Failed"`  
   - Connect Failure Alert Email node output to this node.

10. **Connect Nodes:**  
    - Trigger ➔ HTTP Request  
    - HTTP Request ➔ If Node  
    - If True ➔ Extract Results ➔ Generate HTML Report ➔ Email User ➔ Mark Success  
    - If False ➔ Email IT Alert ➔ Mark Failed

11. **Add Sticky Notes (Optional):**  
    - Add informative sticky notes near nodes explaining their purpose for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                        | Context or Link                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| 🚀 Automated Plagiarism Checker Workflow in n8n - Fully automates plagiarism detection from Google Sheet inputs, sends detailed reports, and logs statuses.       | Workflow summary sticky note at workflow start.                                                              |
| Ensure your RapidAPI key is valid and has quota to avoid API errors.                                                                                               | Critical for the HTTP Request node.                                                                           |
| Replace placeholder emails (`user@test.com`, `it@test.com`, `admin@test.com`) with real addresses or dynamic expressions to handle real users and IT contacts.  | Email Send nodes configuration.                                                                               |
| Google Sheets service account must have edit access to the target spreadsheet to update status columns.                                                           | Applies to Google Sheets Update nodes.                                                                        |
| For enhanced security, rotate API keys regularly and monitor API usage to prevent abuse or quota exhaustion.                                                     | Operational best practice.                                                                                     |
| See RapidAPI Plagiarism Checker documentation for API response schema and rate limits: https://rapidapi.com/                                                        | Useful for troubleshooting API response issues and adapting workflow.                                         |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n integration software. It complies strictly with applicable content policies and contains no illegal, offensive, or protected material. All processed data are legal and public.