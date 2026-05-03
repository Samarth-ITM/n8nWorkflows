Validate and Create LEDGERS Contacts from Google Sheets with Error Handling

https://n8nworkflows.xyz/workflows/validate-and-create-ledgers-contacts-from-google-sheets-with-error-handling-4961


# Validate and Create LEDGERS Contacts from Google Sheets with Error Handling

### 1. Workflow Overview

This workflow automates the validation and creation of contact entries in the LEDGERS system from data sourced in Google Sheets. It continuously monitors a specific Google Sheet for new or updated rows, validates critical contact fields (Name, Email, Mobile), formats the mobile number, attempts contact creation via the LEDGERS API, and handles errors by sending notification emails through Gmail. It also updates a separate Google Sheet with creation timestamps and contact IDs for successfully created contacts.

Logical blocks in the workflow:

- **1.1 Input Reception:** Trigger and initial validation of incoming data rows from Google Sheets.
- **1.2 Contact Field Validation:** Validation of mandatory and format-specific fields (Name, Email, Mobile).
- **1.3 Batch Processing:** Handling multiple contacts in batches to ensure scalable processing.
- **1.4 Mobile Number Formatting:** Normalize mobile numbers to a consistent format.
- **1.5 Contact Creation in LEDGERS:** Create contacts using the LEDGERS API and validate the creation response.
- **1.6 Post-Creation Handling:** Record successful creations in a Google Sheet and notify on failures.
- **1.7 Error Notification:** Send emails for missing names, invalid contact info, or creation errors.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow every minute on changes to a specified Google Sheet and performs initial validation on the presence of the Contact Name.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Contact Name Validation  
  - Contact Name Error Mail Trigger  

- **Node Details:**  

  - **Google Sheets Trigger**  
    - *Type:* Trigger node to detect changes in Google Sheets.  
    - *Configuration:* Polls the sheet every minute; listens to sheet with ID `1m4lbSUudsxNI-thsPBZ1sjMU2yWLSQf0MrURDjBwIlg` and sheet tab with ID `1920008147`.  
    - *Credentials:* Google Sheets OAuth2 account.  
    - *Input/Output:* No input; outputs new/modified rows as JSON.  
    - *Failure Modes:* Auth errors, connection failures, rate limits.  

  - **Contact Name Validation (If node)**  
    - *Type:* Conditional node validating that `Name` field is not empty.  
    - *Configuration:* Checks if `$json['Name']` is non-empty string.  
    - *Input:* From Google Sheets Trigger.  
    - *Outputs:*  
      - True branch: proceeds if Name exists.  
      - False branch: triggers error mail.  
    - *Failure Modes:* Expression evaluation errors if fields missing.  

  - **Contact Name Error Mail Trigger (Gmail node)**  
    - *Type:* Email notification node.  
    - *Configuration:* Sends an email with subject "Contact Creation Failure" and message indicating missing contact name and row number.  
    - *Credentials:* Gmail OAuth2 account.  
    - *Input:* From false branch of Contact Name Validation.  
    - *Failure Modes:* Auth errors, email sending failures.

---

#### 1.2 Contact Field Validation

- **Overview:**  
  Validates the format of Email and Mobile fields, treating empty values as valid. Subsequently, it filters out invalid contacts by condition and handles invalid entries by email notification.

- **Nodes Involved:**  
  - Email & Mobile Format Checker  
  - Email & Mobile Validator  
  - Email/Mobile Error Mail Trigger  

- **Node Details:**  

  - **Email & Mobile Format Checker (Code node)**  
    - *Type:* Code execution node validating Email and Mobile formats with regex.  
    - *Configuration:*  
      - Email regex: standard email format.  
      - Mobile regex: optional country code with `+` prefix and 10 digits.  
      - Empty values considered valid.  
    - *Output:* Adds `validEmail` and `validMobile` boolean fields to JSON.  
    - *Input:* From Contact Name Validation true branch.  
    - *Failure Modes:* JavaScript exceptions if input fields missing or malformed.  

  - **Email & Mobile Validator (If node)**  
    - *Type:* Conditional node checking `validEmail` and `validMobile` are true.  
    - *Input:* From Format Checker.  
    - *Outputs:*  
      - True: proceeds to batch processing.  
      - False: triggers error mail.  
    - *Failure Modes:* Expression errors if `validEmail`/`validMobile` undefined.  

  - **Email/Mobile Error Mail Trigger (Gmail node)**  
    - *Type:* Email notification node.  
    - *Configuration:* Sends email titled "Contact Creation Failure" with message prompting valid Email/Mobile entry for the contact.  
    - *Input:* From false branch of Email & Mobile Validator.  
    - *Failure Modes:* Email sending failures or invalid recipient addresses.

---

#### 1.3 Batch Processing

- **Overview:**  
  Splits the valid contacts into batches for processing, facilitating scalable and manageable execution of subsequent nodes.

- **Nodes Involved:**  
  - Loop Over Items  
  - Replace Me  

- **Node Details:**  

  - **Loop Over Items (SplitInBatches node)**  
    - *Type:* Batch processing node.  
    - *Configuration:* Default options (batch size unspecified, defaults to 1).  
    - *Input:* From Email & Mobile Validator true branch.  
    - *Outputs:*  
      - Batch output to Mobile Formatter and Replace Me nodes.  
    - *Failure Modes:* Improper batch size configurations could stall processing.  

  - **Replace Me (NoOp node)**  
    - *Type:* Placeholder/no-operation node with no processing logic.  
    - *Purpose:* Possibly reserved for future extension or debugging.  
    - *Input:* From Loop Over Items.  
    - *Failure Modes:* None (pass-through).  

---

#### 1.4 Mobile Number Formatting

- **Overview:**  
  Parses and normalizes mobile phone numbers into separate country code and number components, preparing data for LEDGERS API.

- **Nodes Involved:**  
  - Mobile Formatter  

- **Node Details:**  

  - **Mobile Formatter (Code node)**  
    - *Type:* JavaScript code node extracting country code and mobile number.  
    - *Logic:*  
      - Splits raw mobile string on dash `-`.  
      - Ensures country code starts with `+`.  
      - Handles cases with or without dash delimiter.  
    - *Output:* Adds `mobile_country_code` and `mobile` fields to JSON.  
    - *Input:* From Loop Over Items.  
    - *Failure Modes:* Input missing or malformed mobile strings could lead to empty output fields.  

---

#### 1.5 Contact Creation in LEDGERS

- **Overview:**  
  Attempts to create the contact in the LEDGERS system using provided data, then validates the creation response for success or failure.

- **Nodes Involved:**  
  - LEDGERS  
  - Contact Creation Validator  

- **Node Details:**  

  - **LEDGERS (LEDGERS node)**  
    - *Type:* API node for creating contacts in LEDGERS cloud.  
    - *Configuration:*  
      - Contact Name: from `$json['Name']`  
      - Additional fields: email and mobile from `$json['Email']` and `$json['Mobile']`  
    - *Credentials:* LEDGERS API key.  
    - *Input:* From Mobile Formatter.  
    - *Failure Modes:* API auth failure, network errors, invalid data rejection.  

  - **Contact Creation Validator (If node)**  
    - *Type:* Conditional node checks if `errorCode` field does NOT exist in response JSON, indicating success.  
    - *Input:* From LEDGERS node.  
    - *Outputs:*  
      - True: successful creation → proceed to post-creation handling.  
      - False: creation error → trigger error email.  
    - *Failure Modes:* Missing fields in response JSON or unexpected API responses.

---

#### 1.6 Post-Creation Handling

- **Overview:**  
  Records the creation time and contact ID for successfully created contacts and updates a Google Sheet accordingly.

- **Nodes Involved:**  
  - Get Created Time  
  - Update Sheet Contact Creation  

- **Node Details:**  

  - **Get Created Time (Code node)**  
    - *Type:* Adds a timestamp field `created_at` with current date/time formatted as `DD-MM-YYYY HH:mm:ss`.  
    - *Input:* From Contact Creation Validator true branch.  
    - *Output:* JSON enriched with `created_at`.  
    - *Failure Modes:* None significant; system time issues possible but rare.  

  - **Update Sheet Contact Creation (Google Sheets node)**  
    - *Type:* Writes/updates rows in a Google Sheet to log contact creation data.  
    - *Configuration:*  
      - Document ID: `1b1bxz-j9tgUtsl-_1kkoEs30WyF5r3xQ2fd49JmHTwo`  
      - Sheet: tab with gid=0  
      - Columns to update: `Time` (with created_at), `Contact ID`  
      - Matching column: `Contact ID` for upsert operation.  
    - *Credentials:* Google Sheets OAuth2 account.  
    - *Input:* From Get Created Time node.  
    - *Failure Modes:* Auth issues, sheet write conflicts, rate limits.

---

#### 1.7 Error Notification

- **Overview:**  
  Sends email notifications on any detected errors: missing Name, invalid Email/Mobile, or creation failure.

- **Nodes Involved:**  
  - Contact Name Error Mail Trigger  
  - Email/Mobile Error Mail Trigger  
  - Contact Creation Failure Mail Trigger  

- **Node Details:**  

  - **Contact Name Error Mail Trigger (Gmail node)**  
    - *As detailed in block 1.1*  

  - **Email/Mobile Error Mail Trigger (Gmail node)**  
    - *As detailed in block 1.2*  

  - **Contact Creation Failure Mail Trigger (Gmail node)**  
    - *Type:* Email notification node triggered on LEDGERS contact creation failure.  
    - *Configuration:* Sends plain text email with error message and row number.  
    - *Credentials:* Gmail OAuth2 account.  
    - *Input:* From Contact Creation Validator false branch.  
    - *Failure Modes:* Email sending failures, malformed error messages.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                          | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                         |
|-------------------------------|----------------------------------|----------------------------------------|-----------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------|
| Google Sheets Trigger          | Google Sheets Trigger             | Input trigger on Google Sheets changes | None                        | Contact Name Validation            |                                                                                                   |
| Contact Name Validation        | If                               | Validate presence of Contact Name      | Google Sheets Trigger       | Email & Mobile Format Checker, Contact Name Error Mail Trigger |                                                                                                   |
| Contact Name Error Mail Trigger| Gmail                            | Send email if Name missing              | Contact Name Validation     | None                              |                                                                                                   |
| Email & Mobile Format Checker  | Code                             | Validate Email and Mobile format       | Contact Name Validation     | Email & Mobile Validator           |                                                                                                   |
| Email & Mobile Validator       | If                               | Check validEmail and validMobile flags | Email & Mobile Format Checker| Loop Over Items, Email/Mobile Error Mail Trigger |                                                                                                   |
| Email/Mobile Error Mail Trigger| Gmail                            | Email on invalid Email/Mobile          | Email & Mobile Validator    | None                              |                                                                                                   |
| Loop Over Items               | SplitInBatches                   | Batch processing of valid contacts      | Email & Mobile Validator    | Mobile Formatter, Replace Me       |                                                                                                   |
| Replace Me                    | NoOp                             | Placeholder node, no operation          | Loop Over Items             | Loop Over Items                   |                                                                                                   |
| Mobile Formatter             | Code                             | Parse and normalize mobile numbers      | Loop Over Items             | LEDGERS                          |                                                                                                   |
| LEDGERS                      | LEDGERS API Node                 | Create contact in LEDGERS system        | Mobile Formatter            | Contact Creation Validator         |                                                                                                   |
| Contact Creation Validator    | If                               | Validate LEDGERS contact creation result | LEDGERS                    | Get Created Time, Contact Creation Failure Mail Trigger |                                                                                                   |
| Contact Creation Failure Mail Trigger | Gmail                      | Email on LEDGERS contact creation failure | Contact Creation Validator | None                              |                                                                                                   |
| Get Created Time             | Code                             | Add timestamp of contact creation       | Contact Creation Validator  | Update Sheet Contact Creation      |                                                                                                   |
| Update Sheet Contact Creation | Google Sheets                    | Update sheet with creation time and ID | Get Created Time            | None                              |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheets Trigger node:**  
   - Type: Google Sheets Trigger  
   - Configure with your Google Sheets OAuth2 credentials.  
   - Set polling mode to "Every Minute".  
   - Specify the spreadsheet ID: `1m4lbSUudsxNI-thsPBZ1sjMU2yWLSQf0MrURDjBwIlg`.  
   - Specify the sheet tab by ID: `1920008147`.

2. **Add an If node named "Contact Name Validation":**  
   - Condition: Check if `$json['Name']` is not empty (string not empty).  
   - True branch: proceed to next validation.  
   - False branch: send error email.

3. **Add a Gmail node "Contact Name Error Mail Trigger" connected to false branch:**  
   - Credentials: Gmail OAuth2.  
   - Send to: specify recipient(s).  
   - Subject: "Contact Creation Failure".  
   - Message: HTML content indicating missing contact Name and row number:  
     `<p>Contact Name is Missing<br>row no: {{ $json['row_number'] }}</p>`

4. **Add a Code node "Email & Mobile Format Checker" connected to true branch:**  
   - Run once per item.  
   - JavaScript code to validate Email with regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` and Mobile with regex `/^(\+\d{1,3}[- ]?)?\d{10}$/` allowing empty fields as valid.  
   - Output booleans `validEmail` and `validMobile`.

5. **Add an If node "Email & Mobile Validator" connected to Code node:**  
   - Condition: Check that both `$json.validEmail` and `$json.validMobile` are true.  
   - True branch: proceed to batch processing.  
   - False branch: send error email.

6. **Add Gmail node "Email/Mobile Error Mail Trigger" connected to false branch:**  
   - Credentials: Gmail OAuth2.  
   - Subject: "Contact Creation Failure".  
   - Message: HTML content prompting valid Email/Mobile for contact name:  
     `<p>Enter Valid Email/Mobile for contact: <b>{{ $json['Name'] }}</b></p>`

7. **Add SplitInBatches node "Loop Over Items" connected to true branch:**  
   - Default batch size (1 or as preferred).  
   - Output connects to two nodes: "Mobile Formatter" and "Replace Me".

8. **Add a NoOp node "Replace Me" connected to Loop Over Items:**  
   - Purpose: placeholder, no configuration needed.

9. **Add a Code node "Mobile Formatter" connected to Loop Over Items:**  
   - JavaScript to parse `$json.Mobile`:  
     - Split on `-` to extract country code and mobile.  
     - Ensure country code starts with `+`.  
   - Output fields: `mobile_country_code`, `mobile`.

10. **Add LEDGERS API node "LEDGERS" connected to Mobile Formatter:**  
    - Credentials: LEDGERS API key.  
    - Operation: Create contact.  
    - Map fields:  
      - Contact Name: `$json.Name`  
      - Email: `$json.Email`  
      - Mobile: `$json.Mobile`  

11. **Add If node "Contact Creation Validator" connected to LEDGERS node:**  
    - Condition: Check if `$json.errorCode` does *not* exist (contact created successfully).  
    - True branch: proceed to post-creation.  
    - False branch: send error email.

12. **Add Gmail node "Contact Creation Failure Mail Trigger" connected to false branch:**  
    - Credentials: Gmail OAuth2.  
    - Subject: "Contact Creation Failure".  
    - Message (plain text):  
      `Error while creating a contact from the row {{ $json['row_number'] }}\nError: {{ $json['errorMessage'] }}`

13. **Add Code node "Get Created Time" connected to true branch:**  
    - JavaScript adds `created_at` timestamp in `DD-MM-YYYY HH:mm:ss` format.

14. **Add Google Sheets node "Update Sheet Contact Creation" connected to Get Created Time:**  
    - Credentials: Google Sheets OAuth2.  
    - Document ID: `1b1bxz-j9tgUtsl-_1kkoEs30WyF5r3xQ2fd49JmHTwo`.  
    - Sheet by gid=0.  
    - Operation: Append or update rows, matching by `Contact ID`.  
    - Columns updated: `Time` with created_at, `Contact ID`.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The workflow integrates tightly with Google Sheets and Gmail OAuth2 accounts requiring proper credential setup. | Credential setup is mandatory for Google Sheets and Gmail nodes.   |
| The LEDGERS node requires API key credentials and must be configured with correct endpoints.  | LEDGERS API documentation: https://docs.ledgers.io/api              |
| Regex validation in code nodes treats empty Email and Mobile fields as valid by design.       | Adjust regex or validation logic if empty fields should fail.       |
| The placeholder "Replace Me" node is a no-op and can be removed or replaced for custom logic.| Useful for debugging or workflow extension.                         |

---

**Disclaimer:** This workflow analysis is based on an automated n8n workflow export. It complies with content policies and handles only legal and publicly available data.