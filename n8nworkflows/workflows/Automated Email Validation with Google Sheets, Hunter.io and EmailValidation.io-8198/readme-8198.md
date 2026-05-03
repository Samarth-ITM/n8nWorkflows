Automated Email Validation with Google Sheets, Hunter.io and EmailValidation.io

https://n8nworkflows.xyz/workflows/automated-email-validation-with-google-sheets--hunter-io-and-emailvalidation-io-8198


# Automated Email Validation with Google Sheets, Hunter.io and EmailValidation.io

### 1. Workflow Overview

This workflow automates the validation of email addresses entered into a Google Sheet. When a new row containing an email is added, the workflow triggers, validates the email using an external API (EmailValidation.io), and updates the Google Sheet with the validation status. It also includes an alternative validation step using Hunter.io. The workflow is structured in logical blocks that handle input reception, filtering, email validation, and updating results back to the sheet.

**Logical Blocks:**

- **1.1 Input Reception:** Detect new rows added in Google Sheets containing email addresses.
- **1.2 Filtering:** Exclude rows with empty or missing email fields to avoid unnecessary processing.
- **1.3 Email Extraction:** Prepare the email data for validation by isolating the email address.
- **1.4 Email Validation:** Perform email verification using the EmailValidation.io API (primary) and Hunter.io API (alternative).
- **1.5 Result Processing and Update:** Format the validation results and update the Google Sheet with the deliverability status.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow automatically whenever a new row is added to the specified Google Sheet. It monitors the sheet for new data input, which initiates the entire validation process.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**

  - **Google Sheets Trigger**  
    - **Type:** Trigger node that listens for changes in Google Sheets.  
    - **Configuration:**  
      - Triggers on event `rowAdded`.  
      - Polls the sheet every minute to detect new rows.  
      - Targets a specific Google Sheet document and sheet tab via its ID and GID (Sheet1).  
    - **Expressions:** Uses cached document and sheet IDs for efficient referencing.  
    - **Connections:** Outputs to "Filter Empty Cells" node.  
    - **Edge Cases:**  
      - Potential delays due to polling interval (1 minute).  
      - Authentication errors if Google OAuth2 credentials are invalid or expired.  
      - Trigger may miss rows if sheet permissions change or if API limits are exceeded.

#### 2.2 Filtering

- **Overview:**  
  This block filters out rows where the email cell is empty, ensuring that only valid email entries proceed for validation.

- **Nodes Involved:**  
  - Filter Empty Cells

- **Node Details:**

  - **Filter Empty Cells**  
    - **Type:** Filter node that applies conditions on incoming JSON data.  
    - **Configuration:**  
      - Condition: The field `Email` must not be empty (string not empty).  
    - **Expressions:** Evaluates `{{$json.Email}}` to check for emptiness.  
    - **Connections:** Input from Google Sheets Trigger; output to "Take Email Only".  
    - **Edge Cases:**  
      - If column "Email" is misnamed or missing, filter may fail silently.  
      - Case sensitivity enforced (but email field usually case-insensitive).  
      - Empty strings or null values properly filtered out.

#### 2.3 Email Extraction

- **Overview:**  
  This block prepares the data by extracting just the email field from the row to standardize input for the validation API call.

- **Nodes Involved:**  
  - Take Email Only

- **Node Details:**

  - **Take Email Only**  
    - **Type:** Set node used to assign or remap data fields.  
    - **Configuration:**  
      - Sets a new JSON field `Email` equal to the incoming `Email` value.  
    - **Expressions:** `={{ $json.Email }}` ensures the email is passed forward explicitly.  
    - **Connections:** Input from "Filter Empty Cells"; output to "Email Validation API".  
    - **Edge Cases:**  
      - If input data is malformed, the node may pass empty or invalid emails downstream.  
      - No validation here, purely data shaping.

#### 2.4 Email Validation

- **Overview:**  
  This block performs the core email verification using two external services: EmailValidation.io as the primary source and Hunter.io as an alternative.

- **Nodes Involved:**  
  - Email Validation API  
  - Alternative - Hunter for Email Validation

- **Node Details:**

  - **Email Validation API**  
    - **Type:** HTTP Request node to call external REST API.  
    - **Configuration:**  
      - Uses GET request to `https://api.emailvalidation.io/v1/info` endpoint.  
      - API key passed via query parameter `apikey={{api_key}}`.  
      - Email parameter passed via `email={{ $json.Email }}`.  
      - No body or custom headers configured.  
    - **Expressions:** Dynamic URL with expressions for API key and email.  
    - **Connections:** Input from "Take Email Only"; output to "Take Email and Validation Status".  
    - **Edge Cases:**  
      - API key invalid or expired causes authentication errors.  
      - Rate limits from API may cause failures.  
      - Network timeouts or unreachable endpoint.  
      - Email formats not accepted by API may generate errors.  
      - No fallback logic directly connected, but alternative node exists.

  - **Alternative - Hunter for Email Validation**  
    - **Type:** Hunter node configured for email verification.  
    - **Configuration:**  
      - Operation: `emailVerifier` with email parameter `{{$json.Email}}`.  
      - Requires Hunter API credentials (configured separately).  
    - **Connections:** This node is not connected in the current workflow flow but available as an alternative.  
    - **Edge Cases:**  
      - Hunter API limits or invalid credentials.  
      - Email addresses not recognized or with ambiguous status.

#### 2.5 Result Processing and Update

- **Overview:**  
  This block formats the validation responses and updates the Google Sheet by matching emails and writing their deliverability status.

- **Nodes Involved:**  
  - Take Email and Validation Status  
  - Update the Sheets with Validation Status

- **Node Details:**

  - **Take Email and Validation Status**  
    - **Type:** Set node to extract and rename API response fields.  
    - **Configuration:**  
      - Extracts `state` from API response JSON to `status`.  
      - Also maps `email` field from response to a lowercase key.  
    - **Expressions:** `={{ $json.state }}`, `={{ $json.email }}` for dynamic assignment.  
    - **Connections:** Input from "Email Validation API"; output to "Update the Sheets with Validation Status".  
    - **Edge Cases:**  
      - If API response schema changes, fields may be missing.  
      - Null or unexpected values for `state` can lead to incorrect updates.

  - **Update the Sheets with Validation Status**  
    - **Type:** Google Sheets node for updating rows.  
    - **Configuration:**  
      - Operation: `update` rows in the targeted sheet.  
      - Matching column: `Email`.  
      - Columns to update: "Deliverability" column with `status`.  
      - Uses Google OAuth2 credentials.  
      - Sheet and document IDs same as trigger to ensure consistent source.  
    - **Expressions:** Sets values for `Email` and `Deliverability` from previous node outputs.  
    - **Connections:** Input from "Take Email and Validation Status".  
    - **Edge Cases:**  
      - Row matching may fail if email formats differ in case or extra spaces.  
      - Sheet schema changes (column names, deleted columns) can break updates.  
      - OAuth2 token expiration or permission changes may cause update failures.  
      - Concurrent updates to the sheet may cause conflicts or overwrites.

---

### 3. Summary Table

| Node Name                         | Node Type                   | Functional Role                       | Input Node(s)           | Output Node(s)                     | Sticky Note                                                                                              |
|----------------------------------|-----------------------------|-------------------------------------|------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger            | Google Sheets Trigger       | Detect new rows added in Sheet      |                        | Filter Empty Cells                | ## Instructions 1. Take email and other data from the google sheets. 2. If new row added the trigger will trigger the workflow automatically 3. Setup Sheets credentails 4. Setup Email Validation or Hunter Credentials 5. Tadah! You are all good. |
| Filter Empty Cells               | Filter                      | Filter out rows with empty emails   | Google Sheets Trigger  | Take Email Only                  |                                                                                                          |
| Take Email Only                 | Set                         | Extract email field for validation  | Filter Empty Cells     | Email Validation API             |                                                                                                          |
| Email Validation API            | HTTP Request                | Validate email via EmailValidation.io API | Take Email Only        | Take Email and Validation Status |                                                                                                          |
| Alternative - Hunter for Email Validation | Hunter             | Alternative email validation via Hunter.io | (No input connected)   |                                  |                                                                                                          |
| Take Email and Validation Status | Set                         | Map API response fields to status   | Email Validation API   | Update the Sheets with Validation Status |                                                                                                          |
| Update the Sheets with Validation Status | Google Sheets           | Update Sheet with validation status | Take Email and Validation Status |                                  |                                                                                                          |
| Sticky Note                    | Sticky Note                 | Instructional comments               |                        |                                  | ## Instructions 1. Take email and other data from the google sheets. 2. If new row added the trigger will trigger the workflow automatically 3. Setup Sheets credentails 4. Setup Email Validation or Hunter Credentials 5. Tadah! You are all good. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure credentials with Google OAuth2 account having access to your target spreadsheet.  
   - Set event to `rowAdded`.  
   - Set polling interval to every minute (default).  
   - Select the Spreadsheet by pasting its document ID.  
   - Select the Sheet tab (Sheet1 or equivalent) by its GID or name.

2. **Add Filter Node "Filter Empty Cells"**  
   - Add a Filter node connected from Google Sheets Trigger.  
   - Set condition: Field `Email` (case-sensitive) must NOT be empty.  
   - This ensures rows without emails are ignored.

3. **Add Set Node "Take Email Only"**  
   - Connect from Filter node.  
   - Configure to assign a new field `Email` with value `{{$json.Email}}`.  
   - This extracts only the email for the API call.

4. **Add HTTP Request Node "Email Validation API"**  
   - Connect from "Take Email Only".  
   - Set HTTP Method: GET.  
   - URL: `https://api.emailvalidation.io/v1/info?apikey={{api_key}}&email={{ $json.Email }}`  
     - Replace `{{api_key}}` with your EmailValidation.io API key stored in credentials or environment variables.  
   - No body or header parameters needed.  
   - Ensure credentials or environment variables are securely configured.

5. **Add Set Node "Take Email and Validation Status"**  
   - Connect from "Email Validation API".  
   - Assign fields:  
     - `status` = `{{$json.state}}` (validation result from API).  
     - `email` = `{{$json.email}}` (email from API response).  
   - This prepares data for updating the sheet.

6. **Add Google Sheets Node "Update the Sheets with Validation Status"**  
   - Connect from "Take Email and Validation Status".  
   - Configure credentials (Google OAuth2) with write access to the same spreadsheet.  
   - Operation: `update` rows.  
   - Document ID and Sheet Name: same as trigger node.  
   - Matching Columns: set to `Email` to find the row to update.  
   - Map columns:  
     - `Email` → `email` from previous node.  
     - `Deliverability` → `status` from previous node.  
   - Confirm column names match exactly in the sheet.

7. **(Optional) Add Hunter Node "Alternative - Hunter for Email Validation"**  
   - If using Hunter.io as alternative, add this node separately.  
   - Configure with Hunter credentials.  
   - Operation: `emailVerifier`.  
   - Email parameter: `{{$json.Email}}`.  
   - This node is not connected in the current flow but can be integrated with appropriate logic.

8. **Add Sticky Note**  
   - Add a sticky note with instructions for users including setup details for Google Sheets credentials and API keys.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow triggers automatically on new rows added to the Google Sheet, polling every minute for changes.    | Workflow behavior detail                                                                                        |
| Requires Google Sheets OAuth2 credentials with read/write access to the targeted spreadsheet.                    | Credential setup                                                                                                 |
| Requires API key for EmailValidation.io to call the validation endpoint.                                         | https://emailvalidation.io/documentation                                                                        |
| Alternative email validation can be performed via Hunter.io, which requires separate API credentials.            | https://hunter.io/api                                                                                            |
| The sticky note node contains setup instructions and usage tips for end users of the workflow.                   | Visible in workflow interface                                                                                     |

---

**Disclaimer:** The text provided is generated exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal or protected data. All processed data is legal and publicly accessible.