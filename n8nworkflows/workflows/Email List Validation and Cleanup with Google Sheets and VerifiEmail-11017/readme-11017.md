Email List Validation and Cleanup with Google Sheets and VerifiEmail

https://n8nworkflows.xyz/workflows/email-list-validation-and-cleanup-with-google-sheets-and-verifiemail-11017


# Email List Validation and Cleanup with Google Sheets and VerifiEmail

### 1. Workflow Overview

This workflow, titled **Email Subscription Cleaner**, automates the validation and cleanup of email subscriber lists maintained in Google Sheets. It is designed to receive batches of email data via a webhook, normalize and validate each subscriber's email using the VerifiEmail API, classify emails based on validation results and business rules, and then clean up the subscriber list by removing invalid or undesirable entries and appending clean subscribers to a separate sheet. The workflow also returns a clear JSON summary of the processing outcome.

The workflow is logically divided into the following blocks:

- **1.1 Intake & Extraction:** Receives incoming requests for email batch processing and extracts relevant metadata and options.
- **1.2 Subscriber Data Loading & Normalization:** Loads subscriber data from Google Sheets and normalizes it for consistent processing.
- **1.3 Email Validation:** Validates each subscriber’s email using the VerifiEmail API.
- **1.4 Classification:** Classifies emails into categories such as disposable, invalid, role-based, or good.
- **1.5 Cleanup Actions:** Based on classification, deletes invalid entries from the original sheet and appends valid entries to a clean sheet.
- **1.6 Response:** Sends a JSON response summarizing the actions taken per email.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Extraction

- **Overview:**  
  This block handles the receipt of incoming email batch requests via a webhook and extracts key parameters and metadata needed for processing.

- **Nodes Involved:**  
  - Webhook - Receive Email Batch  
  - Extract Inputs

- **Node Details:**

  - **Webhook - Receive Email Batch**  
    - Type: Webhook  
    - Role: Entry point; triggers workflow on receiving a POST request at `/email-batch`.  
    - Configuration: HTTP POST method, path set to "email-batch", response mode set to wait for the last node in the workflow.  
    - Outputs: Passes incoming JSON to Extract Inputs node.  
    - Potential Failures: HTTP timeout, malformed JSON input, missing required fields.

  - **Extract Inputs**  
    - Type: Function  
    - Role: Extracts and structures key input fields (`list_id`, `priority`, `run_notes`, `options`) from the webhook payload.  
    - Configuration: Custom JavaScript that returns an object with those extracted fields from `$json.body`.  
    - Inputs: JSON from webhook.  
    - Outputs: JSON with clean extracted fields for downstream use.  
    - Edge Cases: Missing fields in input JSON may cause undefined values; no explicit error handling.  

- **Sticky Note:**  
  Positioned nearby, explains this block's role clearly:  
  > "Receives cleanup request via Webhook  
  > Extracts listId, priority, run notes & options  
  > Loads subscriber rows from Google Sheets  
  > Provides row_number, email, name, tags, activity  
  > Prepares all data for validation flow"

---

#### 2.2 Subscriber Data Loading & Normalization

- **Overview:**  
  Loads subscriber data from a configured Google Sheet and normalizes email and related fields for uniformity.

- **Nodes Involved:**  
  - Fetch Subscribers  
  - Normalize Subscriber

- **Node Details:**

  - **Fetch Subscribers**  
    - Type: Google Sheets  
    - Role: Reads subscriber data rows from a specified Google Sheet document and sheet tab.  
    - Configuration: Uses OAuth2 credentials for Google Sheets; reads from sheet with `gid=0` (default tab).  
    - Input: Extracted parameters from previous node to identify sheet/document.  
    - Output: Rows of subscriber data including email, name, subscription status, tags, last activity, and notes.  
    - Possible Failures: Authentication errors, API rate limits, missing or misnamed sheet/tab, empty data.

  - **Normalize Subscriber**  
    - Type: Code (JavaScript)  
    - Role: Processes each subscriber row to normalize fields such as email (to lowercase and trimmed), subscription status (boolean), and fills missing optional fields with defaults.  
    - Configuration: Custom JS returning a normalized JSON with keys: `row_number`, `email`, `name`, `subscribed` (boolean), `last_activity`, `tags`, `notes`.  
    - Input: Rows from Google Sheets.  
    - Output: Cleaned subscriber JSON for validation.  
    - Edge Cases: Null or malformed fields, missing subscription status defaults to false.

- **Sticky Note:**  
  > "Normalize subscriber data  
  > Validate email health (MX + disposable)  
  > Mark each email with action → keep/remove"

---

#### 2.3 Email Validation

- **Overview:**  
  Validates subscriber emails using VerifiEmail API to check validity, MX records, and disposability.

- **Nodes Involved:**  
  - Verifi Email

- **Node Details:**

  - **Verifi Email**  
    - Type: VerifiEmail API integration  
    - Role: Validates each email address against VerifiEmail service, returning detailed validation results.  
    - Configuration: Uses VerifiEmail API credentials; input email expression is `{{$json.email}}`.  
    - Input: Normalized subscriber JSON.  
    - Output: Validation details appended to subscriber JSON.  
    - Potential Failures: API authentication failure, rate limiting, network timeouts, invalid email format errors.

---

#### 2.4 Classification

- **Overview:**  
  Classifies each validated email into categories to determine next steps: remove, keep, or tag.

- **Nodes Involved:**  
  - Merge  
  - Classify Email  
  - Should Remove?

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines the results of validation and any other parallel data streams into a single data stream by position.  
    - Configuration: Combine mode with position-based combining.  
    - Inputs: Validation results and normalized subscriber data streams.  
    - Outputs: Single combined JSON per subscriber for classification.  
    - Edge Cases: Mismatched array lengths can cause missing data.

  - **Classify Email**  
    - Type: Code (JavaScript)  
    - Role: Applies classification logic based on VerifiEmail validation results and email pattern matching.  
    - Configuration:  
      - Sets classification and action defaults to "good" and "keep".  
      - If disposable: classification="disposable", action="remove".  
      - If invalid or no MX record: classification="hard_bounce", action="remove".  
      - If role-based email (admin, info, support, help, team): classification="role_account", action="tag".  
    - Outputs enriched JSON with classification and action.  
    - Edge Cases: Case sensitivity in regex, unhandled email formats.

  - **Should Remove?**  
    - Type: If (Conditional)  
    - Role: Routes data based on whether action is "remove".  
    - Configuration: Checks if `$json.action` equals "remove".  
    - Outputs:  
      - True branch: emails to be removed.  
      - False branch: emails to keep or tag.

- **Sticky Note:**  
  > "Classification Logic:  
  > remove → disposable / invalid / role  
  > keep → valid + active  
  > tag → special rule (optional)"

---

#### 2.5 Cleanup Actions

- **Overview:**  
  Performs cleanup operations on the Google Sheet: deletes rows for emails marked for removal and appends clean emails to a separate sheet.

- **Nodes Involved:**  
  - Delete rows or columns from sheet  
  - Append Clean Emails

- **Node Details:**

  - **Delete rows or columns from sheet**  
    - Type: Google Sheets  
    - Role: Deletes the row corresponding to the subscriber marked for removal.  
    - Configuration:  
      - Operation: delete  
      - Sheet and document IDs configured as per source sheet.  
      - `startIndex` set dynamically from `row_number` of subscriber JSON.  
    - Input: Items from "Should Remove?" node true branch.  
    - Outputs: Passes to success response node.  
    - Edge Cases: Row index offset issues (Google Sheets rows are 1-based; verify zero or one-based indexing), concurrent edits, permission issues.

  - **Append Clean Emails**  
    - Type: Google Sheets  
    - Role: Appends cleaned and classified subscriber data to a clean subscriber sheet for record-keeping.  
    - Configuration:  
      - Operation: append  
      - Target sheet and document IDs configured to a designated clean subscribers sheet.  
      - Columns mapped explicitly: name, tags, email, notes, action, last_activity, classification.  
    - Input: Items from "Should Remove?" node false branch.  
    - Outputs: Passes to success response node.  
    - Edge Cases: API quotas, schema mismatch, empty or malformed rows.

- **Sticky Note:**  
  > "Cleanup Actions:  
  > remove → delete row  
  > keep → append to CleanSubscribers  
  > Preserves row_number + metadata"

---

#### 2.6 Response

- **Overview:**  
  Sends a JSON response back to the webhook caller indicating the result of the cleanup operation per email.

- **Nodes Involved:**  
  - Success response

- **Node Details:**

  - **Success response**  
    - Type: Respond to Webhook  
    - Role: Returns JSON response to the webhook caller with status, whether the email was removed, the email address, and notes.  
    - Configuration:  
      - Respond with JSON format.  
      - Response body includes expression:  
        ```json
        {
          "status": "completed",
          "removed": "{{$json.action === 'remove'}}",
          "email": "{{$json.email}}",
          "notes": "cleanup done"
        }
        ```  
    - Input: Final nodes from delete or append operations.  
    - Potential Issues: If the workflow fails before this node, no response will be sent; webhook timeout.

- **Sticky Note:**  
  > "Webhook Output:  
  > Returns: email, action, status  
  > Clean JSON summary for Postman"

---

### 3. Summary Table

| Node Name                       | Node Type              | Functional Role                              | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                 |
|--------------------------------|------------------------|----------------------------------------------|---------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------|
| Webhook - Receive Email Batch   | Webhook                | Receives incoming email batch requests       | None                            | Extract Inputs                         | Webhook triggers workflow whenever a new batch of emails is submitted.                      |
| Extract Inputs                 | Function               | Extracts key parameters from webhook payload | Webhook - Receive Email Batch   | Fetch Subscribers                      | Receives cleanup request via Webhook; Extracts listId, priority, run notes & options        |
| Fetch Subscribers             | Google Sheets          | Reads subscriber rows from Google Sheet      | Extract Inputs                  | Normalize Subscriber                   | Reads subscribers from Google Sheets                                                        |
| Normalize Subscriber          | Code                   | Normalizes subscriber data fields             | Fetch Subscribers               | Verifi Email, Merge                    | Normalize subscriber data; Validate email health (MX + disposable)                         |
| Verifi Email                  | VerifiEmail API        | Validates emails using VerifiEmail service    | Normalize Subscriber            | Merge                                 | Validates email health                                                                    |
| Merge                        | Merge                  | Combines validation and normalized data       | Normalize Subscriber, Verifi Email | Classify Email                    | Combines data streams for classification                                                  |
| Classify Email               | Code                   | Classifies emails for removal, tagging, keep | Merge                          | Should Remove?                        | Classification logic: remove disposable/invalid/role; keep valid; tag special rules        |
| Should Remove?               | If                     | Routes based on remove or keep action          | Classify Email                 | Delete rows or columns from sheet (true), Append Clean Emails (false) |                                                                                             |
| Delete rows or columns from sheet | Google Sheets          | Deletes rows marked for removal                | Should Remove? (true)           | Success response                      | Cleanup: remove → delete row                                                               |
| Append Clean Emails          | Google Sheets          | Appends clean subscriber data                  | Should Remove? (false)          | Success response                      | Cleanup: keep → append to CleanSubscribers                                                 |
| Success response             | Respond to Webhook     | Sends JSON response to webhook caller          | Delete rows or columns from sheet, Append Clean Emails | None                                  | Webhook Output: Returns email, action, status                                              |
| Sticky Note                  | Sticky Note            | Informational notes throughout workflow         | None                          | None                                 | Multiple notes explaining workflow blocks, setup, and logic                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**
   - Type: Webhook  
   - Name: "Webhook - Receive Email Batch"  
   - HTTP Method: POST  
   - Path: `email-batch`  
   - Response Mode: Wait for last node  
   - Purpose: Entry trigger for email batch data.

2. **Create Function Node:**
   - Name: "Extract Inputs"  
   - Type: Function  
   - JavaScript code:
     ```javascript
     return [
       {
         json: {
           listId: $json.body.list_id,
           priority: $json.body.priority,
           runNotes: $json.body.run_notes,
           options: $json.body.options
         }
       }
     ];
     ```
   - Connect from Webhook node.

3. **Create Google Sheets Node (Fetch Subscribers):**
   - Name: "Fetch Subscribers"  
   - Operation: Read rows  
   - Document ID: Your Google Sheets document ID  
   - Sheet Name: Use `gid=0` or your subscribers tab  
   - Credentials: Set up Google Sheets OAuth2 credentials  
   - Connect from Extract Inputs.

4. **Create Code Node (Normalize Subscriber):**
   - Name: "Normalize Subscriber"  
   - Type: Code  
   - JavaScript code:
     ```javascript
     return {
       row_number: $json.row_number,
       email: $json.email.toLowerCase().trim(),
       name: $json.name || "",
       subscribed: $json.subscribed?.toLowerCase() === "yes",
       last_activity: $json.last_activity || null,
       tags: $json.tags || "",
       notes: $json.notes || ""
     };
     ```
   - Connect from Fetch Subscribers.

5. **Create VerifiEmail Node:**
   - Name: "Verifi Email"  
   - Email field: `{{$json.email}}`  
   - Credentials: Set VerifiEmail API credentials  
   - Connect from Normalize Subscriber.

6. **Create Merge Node:**
   - Name: "Merge"  
   - Mode: Combine by position  
   - Connect first input from Verifi Email output  
   - Connect second input from Normalize Subscriber output (if needed)  
   - This merges normalized data and validation results.

7. **Create Code Node (Classify Email):**
   - Name: "Classify Email"  
   - JavaScript code:
     ```javascript
     const email = $json.email;
     const v = $json.details;

     let classification = "good";
     let action = "keep";

     if (v.disposable === true) {
       classification = "disposable";
       action = "remove";
     } else if ($json.valid === false || v.validMxRecord === false) {
       classification = "hard_bounce";
       action = "remove";
     } else if (email.match(/(admin|info|support|help|team)@/i)) {
       classification = "role_account";
       action = "tag";
     }

     return {
       ...$json,
       classification,
       action
     };
     ```
   - Connect from Merge.

8. **Create If Node (Should Remove?):**
   - Name: "Should Remove?"  
   - Condition: `$json.action == "remove"` (string equals)  
   - Connect from Classify Email.

9. **Create Google Sheets Node for Deletion:**
   - Name: "Delete rows or columns from sheet"  
   - Operation: Delete rows  
   - Document ID & Sheet Name: same as subscriber sheet  
   - Start Index: `{{$json.row_number}}`  
   - Credentials: Google Sheets OAuth2  
   - Connect from "Should Remove?" true branch.

10. **Create Google Sheets Node for Appending Clean Emails:**
    - Name: "Append Clean Emails"  
    - Operation: Append  
    - Document ID & Sheet Name: your clean subscriber list sheet/tab  
    - Columns mapping: name, email, tags, notes, action, last_activity, classification mapped from `$('Classify Email').item.json`  
    - Credentials: Google Sheets OAuth2  
    - Connect from "Should Remove?" false branch.

11. **Create Respond to Webhook Node:**
    - Name: "Success response"  
    - Respond with: JSON  
    - Response body:
      ```json
      {
        "status": "completed",
        "removed": "{{$json.action === 'remove'}}",
        "email": "{{$json.email}}",
        "notes": "cleanup done"
      }
      ```
    - Connect from both Delete rows and Append Clean Emails nodes.

12. **Set Credentials:**
    - Google Sheets OAuth2 API credentials with read, delete, append permissions.  
    - VerifiEmail API credentials from verifi.email.

13. **Test the Workflow:**
    - Trigger webhook with a POST JSON payload containing email batch data including `list_id`, `priority`, `run_notes`, `options`.  
    - Verify successful processing, correct classification, deletion, and appending.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow automates email list cleaning by integrating Google Sheets and VerifiEmail API for validation and cleanup.           | General workflow description.                                                                               |
| Required credentials: Google Sheets (read/delete/append), VerifiEmail API key from https://verifi.email.                       | Credential setup.                                                                                           |
| Before running, confirm Google Sheet names and column structure match those expected by the workflow.                          | Setup advice.                                                                                              |
| Tag rules for classification (e.g., role accounts) can be customized in the "Classify Email" code node.                       | Customization tip.                                                                                          |
| The workflow returns clean JSON suitable for use in API clients like Postman.                                                  | Output usage note.                                                                                          |
| See VerifiEmail API docs for rate limits and error handling best practices: https://verifi.email/docs                         | External API resource.                                                                                      |
| Google Sheets API quotas and permissions should be managed to avoid workflow failures during bulk operations.                  | API consideration.                                                                                          |

---

**Disclaimer:**  
The provided text is extracted exclusively from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.