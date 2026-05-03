Flag Bounced Emails in Google Sheets from Gmail Delivery Error Messages

https://n8nworkflows.xyz/workflows/flag-bounced-emails-in-google-sheets-from-gmail-delivery-error-messages-7251


# Flag Bounced Emails in Google Sheets from Gmail Delivery Error Messages

### 1. Workflow Overview

This workflow automates the process of flagging bounced or undelivered email addresses in a Google Sheet based on delivery error messages found in a Gmail account’s Spam folder. Its main purpose is to identify problematic email addresses after an email campaign, mark them in the contact list, and thus prevent future campaign attempts to send to invalid addresses.

The workflow is structured into these logical blocks:

- **1.1 Initialization and Configuration**: Manual trigger and setting the Google Sheet ID.
- **1.2 Gmail Inbox Scanning**: Retrieving all emails from the Spam folder to find delivery error messages.
- **1.3 Delivery Error Filtering**: Filtering emails by subject to isolate undelivered or failure notices.
- **1.4 Email Extraction**: Extracting unique recipient email addresses from the email body text.
- **1.5 Google Sheet Lookup**: Searching the Google Sheet for the extracted email addresses to find corresponding rows.
- **1.6 Update Flag in Google Sheet**: Marking the identified rows with an error flag (`err = "Y"`) in the sheet.
- **1.7 Debugging and Logging**: Nodes used for setting and passing data between steps, plus sticky notes for documentation and debugging.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Configuration

**Overview:**  
This block starts the workflow manually and sets a key configuration parameter: the Google Sheet ID where contacts are stored.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- settings (Set)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually for testing or execution  
  - Configuration: No parameters, triggers the workflow manually  
  - Inputs: None  
  - Outputs: Connects to `settings` node  
  - Edge Cases: None

- **settings**  
  - Type: Set  
  - Role: Defines static workflow parameters (Google Sheet ID)  
  - Configuration: Sets variable `googlesheetid` with a Google Sheet document ID string  
  - Inputs: From Manual Trigger  
  - Outputs: Connects to `readspamfolder`  
  - Edge Cases: Incorrect or missing Sheet ID will cause downstream failures  

---

#### 1.2 Gmail Inbox Scanning

**Overview:**  
This block fetches all emails labeled as Spam in the connected Gmail account to scan for delivery error messages.

**Nodes Involved:**  
- readspamfolder (Gmail)

**Node Details:**

- **readspamfolder**  
  - Type: Gmail  
  - Role: Retrieves all emails from the Gmail Spam folder  
  - Configuration:  
    - Operation: Get all emails  
    - Filters: Label = “SPAM”  
    - Return all results (no pagination)  
  - Credentials: Gmail OAuth2 account specified  
  - Inputs: From `settings`  
  - Outputs: Connects to `undelivered_failure`  
  - Edge Cases:  
    - Gmail API limits or auth failures  
    - Emails not in Spam folder but elsewhere will be missed unless filter changed  
    - Large mailbox might cause timeouts or performance issues  

---

#### 1.3 Delivery Error Filtering

**Overview:**  
Filters the retrieved emails to only pass those whose subject contains keywords indicating delivery failure (e.g., "Undelivered" or "Failure").

**Nodes Involved:**  
- undelivered_failure (If)

**Node Details:**

- **undelivered_failure**  
  - Type: If  
  - Role: Filters emails based on subject content  
  - Configuration:  
    - Condition: Subject contains "Undelivered" OR "Failure"  
    - Combine operation: Any (logical OR)  
  - Inputs: From `readspamfolder`  
  - Outputs: Passes matching emails to `geterremail`  
  - Edge Cases:  
    - Case sensitivity is not explicitly handled (may depend on n8n version)  
    - Emails with different wording may be missed  
    - Empty or missing subject fields may cause no matches  

---

#### 1.4 Email Extraction

**Overview:**  
Extracts email addresses from the body text of the filtered emails and retains a unique email address for further processing.

**Nodes Involved:**  
- geterremail (Code)  
- listerremail (Set)

**Node Details:**

- **geterremail**  
  - Type: Code (JavaScript)  
  - Role: Extracts email addresses from the email body text using regex  
  - Configuration:  
    - Regex pattern matches standard email formats  
    - Removes duplicate emails, retains the first unique email  
    - Returns JSON with a property `extractedEmails` set to the first unique email  
  - Inputs: From `undelivered_failure`  
  - Outputs: Connects to `listerremail`  
  - Edge Cases:  
    - If no emails found, `extractedEmails` will be undefined or empty  
    - Regex may fail on unusual or obfuscated emails  
    - Assumes the text field is named `text` in the input JSON; may need adjustment if source structure changes  

- **listerremail**  
  - Type: Set  
  - Role: Passes along the extracted email in a defined field for subsequent use  
  - Configuration: Assigns the value of `extractedEmails` from previous node into a string field named `extractedEmails`  
  - Inputs: From `geterremail`  
  - Outputs: Connects to `lookupemail`  
  - Edge Cases: None significant  

---

#### 1.5 Google Sheet Lookup

**Overview:**  
Looks up the extracted email address in the Google Sheet to find the matching contact row, retrieving the row number for update.

**Nodes Involved:**  
- lookupemail (Google Sheets)  
- keep_row (Set)

**Node Details:**

- **lookupemail**  
  - Type: Google Sheets  
  - Role: Searches the "contacts" sheet for a row where the "email" column matches the extracted email  
  - Configuration:  
    - Document ID from `settings.googlesheetid`  
    - Sheet name: "contacts"  
    - Lookup value: `{{ $('geterremail').item.json.extractedEmails }}` (dynamic expression)  
    - Lookup column: "email"  
  - Credentials: Google Sheets OAuth2  
  - Inputs: From `listerremail`  
  - Outputs: Connects to `keep_row`  
  - Edge Cases:  
    - No matching row found will result in empty output, may cause later update errors  
    - Google API limits or auth errors  
    - The Google Sheet must contain an "email" column exactly matching the lookup value  

- **keep_row**  
  - Type: Set  
  - Role: Extracts and retains the `row_number` for use in updating the sheet  
  - Configuration: Sets a numeric field `row_number` from the lookup result JSON  
  - Inputs: From `lookupemail`  
  - Outputs: Connects to `update_err`  
  - Edge Cases: None significant  

---

#### 1.6 Update Flag in Google Sheet

**Overview:**  
Updates the identified contact row in the Google Sheet, flagging the `err` column with a value `"Y"` to indicate a bounced or undeliverable email.

**Nodes Involved:**  
- update_err (Google Sheets)

**Node Details:**

- **update_err**  
  - Type: Google Sheets  
  - Role: Updates the row identified by `row_number` in the "contacts" sheet, setting `err = "Y"`  
  - Configuration:  
    - Document ID from `settings.googlesheetid`  
    - Sheet name: "contacts"  
    - Columns updated:  
      - `err` set to `"Y"`  
      - `row_number` used as matching column (key) to locate the row  
    - Mapping mode: Define below  
  - Credentials: Google Sheets OAuth2  
  - Inputs: From `keep_row`  
  - Outputs: None (end of chain)  
  - Edge Cases:  
    - If `row_number` is missing or invalid, update will fail  
    - Permission or API errors from Google Sheets  
    - Concurrent updates may cause conflicts  

---

#### 1.7 Debugging and Documentation Sticky Notes

**Overview:**  
Several sticky note nodes are placed throughout the workflow to provide explanations, instructions, and debugging help.

**Nodes Involved:**  
- Sticky Note7 (main documentation)  
- Sticky Note6 (Google Sheet ID info)  
- Sticky Note (reading spam folder)  
- Sticky Note1 (subject filtering)  
- Sticky Note2 (email extraction)  
- Sticky Note3 (Google Sheet lookup info)  
- Sticky Note4 (email debug output)  
- Sticky Note5 (row number debug output)  
- Sticky Note8 (update step explanation)

**Node Details:**

- Sticky Notes contain detailed instructions, links to prerequisite workflows, Google Sheet templates, and debugging tips.  
- They help users understand the workflow context, required setup, and expected behavior.

---

### 3. Summary Table

| Node Name                   | Node Type        | Functional Role                              | Input Node(s)             | Output Node(s)         | Sticky Note                                                                                          |
|-----------------------------|------------------|----------------------------------------------|---------------------------|------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger   | Starts workflow manually                      | -                         | settings               |                                                                                                    |
| settings                    | Set              | Defines Google Sheet ID                       | When clicking ‘Test workflow’ | readspamfolder         | Define the Google Sheet ID used in this [Workflow](https://n8n.io/workflows/7065-automate-personalized-email-campaigns-with-google-docs-sheets-and-smtp/) |
| readspamfolder              | Gmail            | Reads emails from Gmail Spam folder           | settings                  | undelivered_failure    | Read the folder (Gmail SPAM here), with Mail Delivery issues                                       |
| undelivered_failure         | If               | Filters emails with subjects indicating failure | readspamfolder            | geterremail            | Identify Subject containing "Undelivered" or "Failure"                                             |
| geterremail                 | Code             | Extracts unique email addresses from email body | undelivered_failure       | listerremail           | Extract the "email" from the body message. Unduplication to be sure you get one unique email       |
| listerremail                | Set              | Prepares extracted email for lookup           | geterremail               | lookupemail            |                                                                                                    |
| lookupemail                 | Google Sheets    | Looks up email in Google Sheet contacts        | listerremail              | keep_row               | Find the Row Number in the Google Sheet ID used by this previous [Workflow](https://n8n.io/workflows/7065-automate-personalized-email-campaigns-with-google-docs-sheets-and-smtp/) The `email` column need to match `{{ $('geterremail').item.json.extractedEmails }}` Download the [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing) |
| keep_row                    | Set              | Retains row number for updating                | lookupemail               | update_err             | For debugging. To show the "Row Number" we are going to update in the Google Sheet                 |
| update_err                  | Google Sheets    | Updates Google Sheet row to flag error         | keep_row                  | -                      | Update the Google Sheet and set err = "Y" `row_number` (using to match) `{{ $('lookupemail').item.json.row_number }}` |
| Sticky Note7                | Sticky Note      | Documentation: full workflow explanation       | -                         | -                      | # Scanning Email Inbox for Delivery Errors ... (full content with prerequisites and instructions)  |
| Sticky Note6                | Sticky Note      | Note on Google Sheet ID                         | -                         | -                      | Define the Google  Sheet ID used in this [Workflow](https://n8n.io/workflows/7065-automate-personalized-email-campaigns-with-google-docs-sheets-and-smtp/) |
| Sticky Note                 | Sticky Note      | Note on folder read                             | -                         | -                      | Read the folder (Gmail SPAM here), with Mail Delivery issues                                       |
| Sticky Note1                | Sticky Note      | Note on subject filtering                        | -                         | -                      | Identify Subject containing "Undelivered" or "Failure"                                             |
| Sticky Note2                | Sticky Note      | Note on email extraction                        | -                         | -                      | Extract the "email" from the body message. Unduplication to be sure you get one unique email       |
| Sticky Note3                | Sticky Note      | Note on Google Sheet lookup                      | -                         | -                      | Find the Row Number in the Google  Sheet ID used by this previous [Workflow](https://n8n.io/workflows/7065-automate-personalized-email-campaigns-with-google-docs-sheets-and-smtp/) The `email` column need to match `{{ $('geterremail').item.json.extractedEmails }}` Download the [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing) |
| Sticky Note4                | Sticky Note      | Debug note on extracted emails                  | -                         | -                      | For debugging. To show the emails in error                                                         |
| Sticky Note5                | Sticky Note      | Debug note on row number                         | -                         | -                      | For debugging. To show the "Row Number" we are going to update in the Google Sheet                 |
| Sticky Note8                | Sticky Note      | Note on updating Google Sheet                    | -                         | -                      | Update the Google Sheet and set err = "Y" `row_number` (using to match) `{{ $('lookupemail').item.json.row_number }}` |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No special parameters

2. **Create Set Node for Settings**  
   - Name: `settings`  
   - Type: Set  
   - Add a string field `googlesheetid` with your Google Sheet document ID (e.g., `1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U`)  
   - Connect `When clicking ‘Test workflow’` → `settings`

3. **Create Gmail Node to Read Spam Folder**  
   - Name: `readspamfolder`  
   - Type: Gmail  
   - Operation: Get All Emails  
   - Set filters → Label IDs: `SPAM`  
   - Return All: true  
   - Connect `settings` → `readspamfolder`  
   - Set Gmail OAuth2 credentials

4. **Create If Node to Filter Delivery Failures**  
   - Name: `undelivered_failure`  
   - Type: If  
   - Condition:  
     - Check if `subject` contains "Undelivered" OR contains "Failure"  
     - Combine operation: Any  
   - Connect `readspamfolder` → `undelivered_failure`

5. **Create Code Node to Extract Emails**  
   - Name: `geterremail`  
   - Type: Code (JavaScript)  
   - Code snippet (JS): Use regex to find email addresses in the email body text property (adjust if field not `text`):  
     ```js
     const emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g;
     const removeDuplicates = (arr) => [...new Set(arr)];
     return $input.all().map(item => {
       const text = item.json.text || "";
       const emails = text.match(emailRegex) || [];
       const uniqueEmails = removeDuplicates(emails);
       return { json: { ...item.json, extractedEmails: uniqueEmails[0] } };
     });
     ```  
   - Connect `undelivered_failure` → `geterremail`  
   - Set Execute Once = true

6. **Create Set Node to Pass Extracted Email**  
   - Name: `listerremail`  
   - Type: Set  
   - Add string field `extractedEmails` with value `={{ $json.extractedEmails }}`  
   - Connect `geterremail` → `listerremail`

7. **Create Google Sheets Lookup Node**  
   - Name: `lookupemail`  
   - Type: Google Sheets  
   - Operation: Lookup  
   - Document ID: `={{ $('settings').item.json.googlesheetid }}`  
   - Sheet Name: `contacts`  
   - Lookup Column: `email`  
   - Lookup Value: `={{ $json.extractedEmails }}` or `={{ $('geterremail').item.json.extractedEmails }}` (adjust accordingly)  
   - Connect `listerremail` → `lookupemail`  
   - Set Google Sheets OAuth2 credentials  
   - Execute Once: false

8. **Create Set Node to Keep Row Number**  
   - Name: `keep_row`  
   - Type: Set  
   - Add number field `row_number` with value `={{ $json.row_number }}`  
   - Connect `lookupemail` → `keep_row`

9. **Create Google Sheets Update Node**  
   - Name: `update_err`  
   - Type: Google Sheets  
   - Operation: Update  
   - Document ID: `={{ $('settings').item.json.googlesheetid }}`  
   - Sheet Name: `contacts`  
   - Matching Column: `row_number`  
   - Columns to update:  
     - `err`: Set to `"Y"`  
     - `row_number`: Pass through from previous node  
   - Connect `keep_row` → `update_err`  
   - Set Google Sheets OAuth2 credentials

10. **Add Sticky Note Nodes (Optional but Recommended)**  
    - Add sticky notes at relevant points describing purpose, usage instructions, and debugging tips. Use content from the original workflow’s sticky notes for full documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow requires a prior email campaign workflow such as [Automate Personalized Email Campaigns with Google Docs, Sheets, and SMTP](https://n8n.io/workflows/7065-automate-personalized-email-campaigns-with-google-docs-sheets-and-smtp/).                                                                                                                                                                | Prerequisite workflow for sending emails before running this error scanning workflow                                  |
| Use the Google Sheet template provided here for contact management: [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing)                                                                                                                                                                                                             | Template for the contacts sheet structure used in the workflow                                                        |
| Gmail API requires OAuth2 credentials; ensure your Gmail OAuth2 credentials allow reading emails and labels including Spam.                                                                                                                                                                                                                                                                               | Credential setup for Gmail node                                                                                        |
| Google Sheets API requires OAuth2 credentials with permission to read and update the specified document.                                                                                                                                                                                                                                                                                                  | Credential setup for Google Sheets nodes                                                                               |
| Tested on n8n version 1.105.2 running on Ubuntu; behavior may vary slightly on other versions or platforms.                                                                                                                                                                                                                                                                                               | Version-specific requirements                                                                                          |
| For help and community support, visit the [n8n Forum](https://community.n8n.io/) or contact the workflow author on [LinkedIn](https://www.linkedin.com/in/stephaneheckel/).                                                                                                                                                                                                                               | Support and contact information                                                                                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.