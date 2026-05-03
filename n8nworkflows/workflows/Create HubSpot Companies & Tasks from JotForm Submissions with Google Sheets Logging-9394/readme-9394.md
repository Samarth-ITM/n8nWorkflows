Create HubSpot Companies & Tasks from JotForm Submissions with Google Sheets Logging

https://n8nworkflows.xyz/workflows/create-hubspot-companies---tasks-from-jotform-submissions-with-google-sheets-logging-9394


# Create HubSpot Companies & Tasks from JotForm Submissions with Google Sheets Logging

### 1. Workflow Overview

This workflow automates the processing of form submissions from JotForm to create and manage company records and associated tasks in HubSpot, while maintaining an audit log in Google Sheets. It is designed to streamline lead capture, contact management, and task assignment for marketing queries submitted via a web form.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception (JotForm Trigger):** Captures new submissions from a specific JotForm form.
- **1.2 HubSpot Company Creation & Data Formatting:** Creates or updates company records in HubSpot and formats raw form data into a structured format for downstream processing.
- **1.3 HubSpot Task Creation:** Generates follow-up tasks in HubSpot linked to the newly created or existing company.
- **1.4 Domain Update & Post-Processing:** Updates the domain property of the HubSpot company records after task creation.
- **1.5 Logging:** Records all processed data and API responses into a Google Sheets document for auditing and monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception (JotForm Trigger)

**Overview:**  
This block listens for new submissions on a specific JotForm form. When a submission occurs, it captures relevant fields and triggers the workflow.

**Nodes Involved:**  
- JotForm Trigger

**Node Details:**

- **JotForm Trigger**  
  - *Type:* JotForm Trigger (Event Trigger)  
  - *Configuration:* Monitors form with ID `252808415357461` for new submissions. Uses stored JotForm API credentials.  
  - *Key Expressions:* None (event-driven)  
  - *Input:* External trigger from JotForm webhook  
  - *Output:* JSON payload containing submitted form fields such as first and last name, email, LinkedIn profile, company name, marketing budget, domain, and specific query.  
  - *Edge Cases:*  
    - Failure if JotForm API credentials expire or webhook is misconfigured.  
    - Missing or malformed submission data may affect downstream processing.  
  - *Sticky Note:* Provides summary of captured fields and workflow starting point.  

---

#### 2.2 HubSpot Company Creation & Data Formatting

**Overview:**  
Creates a HubSpot company record based on the submitted company name (if not existing), then formats all raw JotForm data into a clean, structured JSON object to standardize information for further processing.

**Nodes Involved:**  
- Create a company1 (HubSpot)  
- Formating Data (Code)

**Node Details:**

- **Create a company1**  
  - *Type:* HubSpot node (Company resource)  
  - *Configuration:* Creates a company in HubSpot using the submitted company name (`{{$json.CompanyName}}`). Uses HubSpot API credentials.  
  - *Key Expressions:* Company name pulled from input JSON.  
  - *Input:* Triggered by JotForm Trigger output.  
  - *Output:* HubSpot company data including newly generated company ID.  
  - *Edge Cases:*  
    - Duplicate company creation if company deduplication is not enabled in HubSpot.  
    - API errors due to authentication or rate limiting.  
  - *Always Output Data:* Ensures downstream nodes receive output even if creation partially fails.

- **Formating Data**  
  - *Type:* Code (JavaScript)  
  - *Configuration:* Processes all input data by:  
    - Unescaping quotes in text fields.  
    - Extracting and sanitizing relevant fields (first name, last name, full name, email, LinkedIn profile, company name, marketing budget, domain, specific query).  
    - Generating a formatted HubSpot task title.  
    - Assigning HubSpot owner IDs based on company name hash logic (even/odd length).  
    - Creating a timestamp 24 hours in the future for task scheduling.  
    - Returning a structured JSON array with these fields.  
  - *Key Expressions:* Uses input data from previous node; creates new JSON structure.  
  - *Input:* Output from Create a company1 node.  
  - *Output:* Array of formatted JSON objects with clean and enriched submission data.  
  - *Edge Cases:*  
    - Malformed input data could cause errors in string operations.  
    - Owner ID assignment depends on company name length, which may be unpredictable.  

- *Sticky Note:* Provides explanation of company creation and data formatting purpose, including field sanitation and task title generation.

---

#### 2.3 HubSpot Task Creation

**Overview:**  
Creates a task in HubSpot for each formatted submission, associating it with the corresponding company record. The task includes detailed information extracted from the form and assigns priority, status, type, and owner automatically.

**Nodes Involved:**  
- Wait  
- Create HubSpot Task

**Node Details:**

- **Wait**  
  - *Type:* Wait  
  - *Configuration:* Waits for 10 seconds before proceeding to prevent API rate limit issues.  
  - *Input:* Data from Formating Data node.  
  - *Output:* Delayed data forwarded to Create HubSpot Task.  
  - *Edge Cases:* None significant; used for throttling.

- **Create HubSpot Task**  
  - *Type:* HTTP Request  
  - *Configuration:*  
    - Sends POST request to HubSpot Tasks API endpoint.  
    - Constructs JSON body with task properties populated from formatted data, including subject, body (detailed multi-line string with company, person, email, LinkedIn, domain, budget, and query), status (NOT_STARTED), priority (HIGH), type (CALL), timestamp, and owner ID.  
    - Associates the task with the company ID from formatted data.  
    - Uses bearer token authorization with a personal access token (PAT).  
    - Sets headers for JSON content type.  
  - *Key Expressions:* Uses expressions to safely fallback to default values when fields are missing.  
  - *Input:* Output from Wait node.  
  - *Output:* JSON response from HubSpot API call (task creation result).  
  - *Error Handling:* Configured to continue workflow even if errors occur to avoid blocking downstream nodes.  
  - *Edge Cases:*  
    - API rate limiting or authorization failures.  
    - Missing company ID may cause task association failure.  
  - *Sticky Note:* Explains logic for task creation, rate limiting prevention, and task linking.

---

#### 2.4 Domain Update & Post-Processing

**Overview:**  
Updates the HubSpot company record to set the `domain` property based on the data submitted in the form, ensuring company profiles include website information.

**Nodes Involved:**  
- Wait10  
- Set Company Domain

**Node Details:**

- **Wait10**  
  - *Type:* Wait  
  - *Configuration:* Default wait time (default parameters not explicitly set, likely immediate pass-through or short delay) to ensure proper sequencing after task creation.  
  - *Input:* Output from Create HubSpot Task node.  
  - *Output:* Passes data to Set Company Domain node.  
  - *Edge Cases:* None significant.

- **Set Company Domain**  
  - *Type:* HTTP Request  
  - *Configuration:*  
    - Sends PATCH request to HubSpot Companies API endpoint for the company ID obtained from previous node.  
    - Updates the `domain` property with the submitted domain.  
    - Uses bearer token authorization with PAT and JSON content-type headers.  
  - *Key Expressions:* Company ID and domain dynamically extracted via expressions.  
  - *Input:* Output from Wait10 node.  
  - *Output:* JSON response from HubSpot API company update call.  
  - *Edge Cases:*  
    - Missing or invalid company ID causes failure.  
    - API authorization or rate limiting errors.  
  - *Sticky Note:* Describes the intent to ensure domain field is properly set and to prevent duplicates.

---

#### 2.5 Logging

**Overview:**  
Logs all processed submissions and their corresponding HubSpot API responses into a Google Sheet for audit and monitoring purposes.

**Nodes Involved:**  
- Storel Logs (Google Sheets)

**Node Details:**

- **Storel Logs**  
  - *Type:* Google Sheets (Append or Update)  
  - *Configuration:*  
    - Appends or updates rows in a specific Google Sheet and tab named "AlreadyExistingHC".  
    - Columns logged include Notes (with created and updated timestamps), domain, company name, and HubSpot company ID.  
    - Matching is performed on the domain column to avoid duplicates.  
    - Uses OAuth2 credentials for Google Sheets access.  
  - *Key Expressions:* Uses JSON paths to extract relevant fields for logging.  
  - *Input:* Output from Set Company Domain node.  
  - *Output:* Confirmation of sheet update.  
  - *Edge Cases:*  
    - Google Sheets API quota limits or credential expiration.  
    - Mismatched domains causing duplicate entries.  
  - *Sticky Note:* Emphasizes importance of logging for transparency and debugging.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                              | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                     |
|---------------------|----------------------|----------------------------------------------|-----------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| JotForm Trigger     | JotForm Trigger       | Captures form submissions                      | —                           | Create a company1       | Starts workflow; captures form fields like Name, Email, LinkedIn, Company, Budget, Domain, Query. |
| Create a company1   | HubSpot               | Creates company record in HubSpot              | JotForm Trigger             | Formating Data          | Creates HubSpot company if new; uses company name from form submission.                         |
| Formating Data      | Code                  | Formats and sanitizes form data                 | Create a company1            | Wait                    | Standardizes data; assigns owners; generates task titles and timestamps.                        |
| Wait                | Wait                  | Rate limit throttle before task creation       | Formating Data              | Create HubSpot Task     | Waits 10 seconds to avoid API rate limits.                                                     |
| Create HubSpot Task | HTTP Request          | Creates follow-up task in HubSpot               | Wait                        | Wait10                  | Creates task with detailed info; associates with company; sets priority and owner.             |
| Wait10              | Wait                  | Sequencing delay before updating company domain | Create HubSpot Task          | Set Company Domain      | Ensures proper sequence for domain update.                                                     |
| Set Company Domain  | HTTP Request          | Updates domain property of HubSpot company      | Wait10                      | Storel Logs             | Updates company domain in HubSpot to enrich company profile.                                   |
| Storel Logs         | Google Sheets         | Logs processed submissions and API results      | Set Company Domain           | —                       | Appends/updates audit trail in Google Sheets for transparency and debugging.                   |
| Sticky Note         | Sticky Note           | Documentation                                   | —                           | —                       | Explains JotForm Trigger block and captured fields.                                           |
| Sticky Note1        | Sticky Note           | Documentation                                   | —                           | —                       | Explains HubSpot Company Creation & Data Formatting block.                                   |
| Sticky Note2        | Sticky Note           | Documentation                                   | —                           | —                       | Explains HubSpot Task Creation block and logic.                                               |
| Sticky Note4        | Sticky Note           | Documentation                                   | —                           | —                       | Explains Loop Over & Domain Setting block and update logic.                                  |
| Sticky Note5        | Sticky Note           | Documentation                                   | —                           | —                       | Explains Logging block and importance of audit trails.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node:**  
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials.  
   - Set to listen to form ID `252808415357461`.  
   - Position near the workflow start.

2. **Add HubSpot Company Create Node:**  
   - Type: HubSpot node (resource: company, operation: create).  
   - Use HubSpot API credentials.  
   - Set company name field to expression: `={{ $json.CompanyName }}` or equivalent field from JotForm submission.  
   - Connect output of JotForm Trigger to this node.

3. **Add Code Node to Format Data:**  
   - Type: Code (JavaScript).  
   - Paste provided JS code that:  
     - Unescapes quotes in strings.  
     - Extracts and normalizes first name, last name, email, LinkedIn, company name, marketing budget, domain, and specific query.  
     - Creates full name and task title string.  
     - Assigns owner ID based on company name length parity.  
     - Sets task timestamp 24 hours in the future.  
     - Returns array of formatted JSON objects.  
   - Connect output of HubSpot Company Create node to this node.

4. **Add Wait Node Before Task Creation:**  
   - Type: Wait  
   - Configure with 10 seconds delay to prevent API rate limit issues.  
   - Connect output of formatting code node to this wait node.

5. **Add HTTP Request Node to Create HubSpot Task:**  
   - Type: HTTP Request  
   - Set method to POST.  
   - URL: `https://api.hubapi.com/crm/v3/objects/tasks`  
   - Headers:  
     - Authorization: `Bearer <your personal access token>`  
     - Content-Type: `application/json`  
   - JSON body: Construct task JSON including:  
     - `hs_task_subject` from formatted data task title.  
     - `hs_task_body`: multiline string concatenating company, person, email, LinkedIn, domain, budget, query.  
     - `hs_task_status`: `"NOT_STARTED"`  
     - `hs_task_priority`: `"HIGH"`  
     - `hs_task_type`: `"CALL"`  
     - `hs_timestamp`: task timestamp from formatted data.  
     - `hubspot_owner_id`: assigned owner ID.  
     - Associations linking to the company ID.  
   - Configure to continue on error to avoid halting workflow.  
   - Connect output of Wait node to this node.

6. **Add Second Wait Node (Wait10):**  
   - Type: Wait  
   - Leave default or short delay to ensure proper sequencing.  
   - Connect output of Create HubSpot Task node to this node.

7. **Add HTTP Request Node to Update Company Domain:**  
   - Type: HTTP Request  
   - Method: PATCH  
   - URL: `https://api.hubapi.com/crm/v3/objects/companies/{{ company_id }}` (use expression to get company ID from previous nodes)  
   - Headers:  
     - Authorization: `Bearer <your personal access token>`  
     - Content-Type: `application/json`  
   - JSON body: Set `"domain"` property from form data.  
   - Connect output of Wait10 node to this node.

8. **Add Google Sheets Node to Log Data:**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Configure with OAuth2 credentials for Google Sheets API.  
   - Document ID: Set to your specific Google Sheet ID.  
   - Sheet name: Use the tab dedicated for logs (e.g., "AlreadyExistingHC").  
   - Map columns:  
     - Notes: `This was created at {{ $json.createdAt }} & was updated at {{ $json.updatedAt }}`  
     - Domain, Company, HubspotCompanyID: mapped from the HubSpot API response.  
   - Configure matching column as domain to avoid duplicates.  
   - Connect output of Set Company Domain node to this node.

9. **Add Sticky Notes for Documentation (Optional):**  
   - Add multiple Sticky Note nodes describing each logical block for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow begins with JotForm submissions capturing key marketing query data fields.                                  | See Sticky Note at start of workflow for detailed fields captured.                                 |
| HubSpot owner assignment logic is based on company name length parity (even/odd). Replace with your own logic if needed. | Embedded in the "Formating Data" code node.                                                        |
| Use personal access tokens (PAT) for HubSpot API authorization in HTTP Request nodes.                                | Ensure your PAT has sufficient scopes for CRM objects and tasks API access.                         |
| Rate limiting is managed by inserting Wait nodes before API calls to HubSpot.                                        | Wait configured typically as 10 seconds to reduce 429 errors.                                      |
| Google Sheets logging provides audit trail for all processed submissions and API interactions.                       | Helpful for monitoring and debugging workflow executions.                                          |
| For further HubSpot API documentation, visit: https://developers.hubspot.com/docs/api/crm/companies                  | Official HubSpot API docs for companies and tasks.                                                |
| For JotForm webhook setup details, see: https://www.jotform.com/integrations/n8n/                                    | JotForm integration with n8n for webhook and API usage.                                            |
| Google Sheets API authorization uses OAuth2; refresh tokens may be required for long-running workflows.               | See Google Sheets OAuth docs for setup details.                                                    |

---

**Disclaimer:**  
The text provided reflects an automated workflow created using n8n, an integration and automation tool. It complies strictly with applicable content policies and contains no illegal or protected elements. All processed data are legal and public.