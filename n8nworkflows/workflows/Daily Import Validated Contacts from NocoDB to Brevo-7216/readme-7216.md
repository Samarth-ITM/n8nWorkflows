Daily Import Validated Contacts from NocoDB to Brevo

https://n8nworkflows.xyz/workflows/daily-import-validated-contacts-from-nocodb-to-brevo-7216


# Daily Import Validated Contacts from NocoDB to Brevo

### 1. Workflow Overview

This workflow automates the daily import of newly validated contacts from a NocoDB database into the Brevo (formerly Sendinblue) contact management platform. It targets use cases where an organization maintains a contacts database in NocoDB and wants to synchronize validated contacts to Brevo for marketing or communication purposes, ensuring only complete and valid contacts are imported.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow automatically once per day.
- **1.2 Retrieve New Contacts**: Fetches contacts from NocoDB marked as not yet imported.
- **1.3 Validation Checks**: Ensures required fields are populated and emails are not disposable.
- **1.4 Batch Processing**: Processes contacts one by one to limit system load.
- **1.5 Contact Creation in Brevo**: Adds validated contacts to Brevo.
- **1.6 Status Updates in NocoDB**: Updates the status of contacts in NocoDB to track processing stages and outcomes.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview**: Automatically starts the workflow once a day at a scheduled time.
- **Nodes Involved**:  
  - Schedule Trigger  
  - Sticky Note (description)

- **Node Details**:

  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Role: Entry point for the workflow to run on a daily schedule (interval set to daily).  
    - Configuration: Runs automatically once per day. No additional parameters set beyond the default daily interval.  
    - Input: None (trigger node)  
    - Output: Passes trigger data to "NocoDB: Get new users list" node.  
    - Edge Cases: If the n8n instance is down at scheduled time, the trigger will miss execution; no retry logic here.  
    - Version: 1.2  

  - **Sticky Note**  
    - Purpose: Documents the scheduled trigger purpose.  
    - Content: "Runs the workflow automatically once a day at the scheduled time."

#### 2.2 Retrieve New Contacts

- **Overview**: Queries the NocoDB project to get all contacts with status `0-not-imported` indicating new users to be processed.
- **Nodes Involved**:  
  - NocoDB: Get new users list  
  - Sticky Note (description)

- **Node Details**:

  - **NocoDB: Get new users list**  
    - Type: `NocoDB` node  
    - Role: Retrieves all records from the table `m5mxqjs3ae68oyc` where status equals `0-not-imported`.  
    - Configuration:  
      - Operation: `getAll`  
      - Table: `m5mxqjs3ae68oyc`  
      - Filter: `(status,eq,0-not-imported)`  
      - Project ID: `p4lnw5vwzf2yy3i`  
      - Authentication: API token credentials (secured)  
    - Input: Trigger data from Schedule Trigger  
    - Output: JSON array of new contacts to process  
    - Edge Cases: API token expiration or network errors could cause failure. Empty results if no new users.  
    - Version: 3  

  - **Sticky Note1**  
    - Content: "Fetches all new user records from the NocoDB table where the status is 0-not-imported. This status marks records that have not yet been processed by the flow (new users)."

#### 2.3 Validation Checks

- **Overview**: Validates that each contact has necessary fields (first name, last name, email) filled and that the email is not a disposable or temporary address. Updates the status in NocoDB if validation fails.
- **Nodes Involved**:  
  - Check if parameters are not empty (If node)  
  - Check if email is Disposal (If node)  
  - NocoDB: change status to 1-empty-fields  
  - NocoDB: change status to 2-disposal-email  
  - Sticky Notes (description)

- **Node Details**:

  - **Check if parameters are not empty**  
    - Type: `If` node  
    - Role: Checks that `first_name`, `last_name`, and `email` fields are all non-empty.  
    - Configuration:  
      - Version 2 conditions, strict string validation, case sensitive  
      - Conditions combined with AND  
      - Operators: `notEmpty` for each field  
      - Expressions:  
        - `$json.first_name`  
        - `$json.last_name`  
        - `$json.email`  
    - Input: Contacts data from NocoDB  
    - Output:  
      - True path: proceeds to email validation  
      - False path: updates contact status to `1-empty-fields` in NocoDB  
    - Edge Cases: Missing fields, null values, or unexpected JSON structure can cause expression failures.  
    - Version: 2.2  

  - **Check if email is Disposal**  
    - Type: `If` node  
    - Role: Verifies email is not from a known disposable email provider by regex pattern matching.  
    - Configuration:  
      - Regex negative match against a large pattern matching disposable email domains and keywords (e.g., temp, yopmail, mailinator, protonmail, etc.)  
      - Expression: `$json.email`  
    - Input: Contacts passing previous validation  
    - Output:  
      - True path: proceeds to batch processing  
      - False path: updates contact status to `2-disposal-email` in NocoDB  
    - Edge Cases: Regex complexity might cause performance issues or false negatives/positives.  
    - Version: 2.2  

  - **NocoDB: change status to 1-empty-fields**  
    - Type: `NocoDB` node  
    - Role: Updates the contact record status to indicate missing required fields.  
    - Configuration:  
      - Operation: `update`  
      - Table: `m5mxqjs3ae68oyc`  
      - Fields updated: `status` = `1-empty-fields`, `id` from current item  
      - Project ID and authentication as before  
    - Input: Contacts failing first validation  
    - Output: None (ends for those records)  
    - Edge Cases: API failures or concurrency issues when updating.  
    - Version: 3  

  - **NocoDB: change status to 2-disposal-email**  
    - Type: `NocoDB` node  
    - Role: Updates the contact record status to indicate use of disposable email address.  
    - Configuration: As above, but setting `status` = `2-disposal-email`  
    - Input: Contacts failing disposable email check  
    - Output: None  
    - Edge Cases: Same as above  
    - Version: 3  

  - **Sticky Note2**  
    - Content: "Verifies that the required fields: first_name, last_name & email are all filled in before continuing. Updates the record’s status to 1-empty-fields in NocoDB for entries missing any required information."

  - **Sticky Note3**  
    - Content: "Verifies that the email address is not from a known disposable or temporary email provider before continuing. We don't want to send email to disposal email addresses. Updates the record’s status to 2-disposal-email in NocoDB for entries identified as using disposable or temporary email addresses."

#### 2.4 Batch Processing

- **Overview**: Processes contacts one at a time to avoid heavy system load while creating contacts in Brevo.
- **Nodes Involved**:  
  - Loop Over Items (`SplitInBatches` node)  
  - Sticky Note (description)

- **Node Details**:

  - **Loop Over Items**  
    - Type: `SplitInBatches` node  
    - Role: Splits the input array of contacts into individual items (batch size defaults to 1) for sequential processing.  
    - Configuration: Default options (batch size 1).  
    - Input: Contacts passing all validations  
    - Output: Sends one contact per cycle to "Brevo: Create Contact" node  
    - Edge Cases: Large datasets will take longer to process; node must be monitored for timeouts or failures.  
    - Version: 3  

  - **Sticky Note4**  
    - Content: "Runs one record at a time to avoid heavy load from creating contacts in bulk."

#### 2.5 Contact Creation in Brevo

- **Overview**: Creates a new contact in Brevo with the validated first name, last name, and email address.
- **Nodes Involved**:  
  - Brevo: Create Contact  
  - Sticky Note (description)

- **Node Details**:

  - **Brevo: Create Contact**  
    - Type: `SendInBlue` node (Brevo API integration)  
    - Role: Creates a contact in Brevo with email and attributes for first and last name.  
    - Configuration:  
      - Resource: `contact`  
      - Email: Expression `$json.email`  
      - Attributes:  
        - FIRSTNAME: `$json.first_name`  
        - LASTNAME: `$json.last_name`  
      - Credentials: Brevo API key (secured)  
    - Input: Single contact from batch node  
    - Output: Result of contact creation  
    - Edge Cases: API limits, invalid API key, network errors, duplicate contact handling by Brevo.  
    - Version: 1  

  - **Sticky Note5**  
    - Content: "Creates a new contact in Brevo using the provided first_name, last_name, and email."

#### 2.6 Status Updates in NocoDB After Contact Creation

- **Overview**: Updates the original contact record status to `3-contact-created` in NocoDB after successful creation in Brevo.
- **Nodes Involved**:  
  - NocoDB: change status to 3-contact-created  
  - Sticky Note (description)

- **Node Details**:

  - **NocoDB: change status to 3-contact-created**  
    - Type: `NocoDB` node  
    - Role: Updates the status of the processed contact to indicate successful import.  
    - Configuration:  
      - Table: `m5mxqjs3ae68oyc`  
      - Operation: `update`  
      - Fields:  
        - `id`: from the current item (`$('Loop Over Items').item.json.Id`)  
        - `status`: `3-contact-created`  
      - Project ID and authentication as before  
    - Input: Output from Brevo node  
    - Output: Loops back to "Loop Over Items" node to process next record  
    - Edge Cases: Update failure due to API issues or concurrency may cause inconsistent status.  
    - Version: 3  

  - **Sticky Note6**  
    - Content: "Updates the record’s status to 3-contact-created in NocoDB after the contact is successfully added to Brevo."

---

### 3. Summary Table

| Node Name                          | Node Type               | Functional Role                            | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                                  |
|-----------------------------------|-------------------------|--------------------------------------------|-----------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                  | Schedule Trigger        | Triggers workflow daily                     | None                        | NocoDB: Get new users list        | Runs the workflow automatically once a day at the scheduled time.                                          |
| NocoDB: Get new users list        | NocoDB                  | Fetch new contacts with status 0-not-imported | Schedule Trigger            | Check if parameters are not empty | Fetches all new user records from the NocoDB table where the status is 0-not-imported.                       |
| Check if parameters are not empty | If                      | Validate required fields are filled         | NocoDB: Get new users list   | Check if email is Disposal; NocoDB: change status to 1-empty-fields | Verifies that the required fields: first_name, last_name & email are all filled in before continuing. Updates the record’s status to 1-empty-fields in NocoDB for entries missing any required information. |
| NocoDB: change status to 1-empty-fields | NocoDB                  | Mark records missing required fields         | Check if parameters are not empty (false branch) | None                             |                                                                                                              |
| Check if email is Disposal        | If                      | Validate email is not disposable            | Check if parameters are not empty (true branch) | Loop Over Items; NocoDB: change status to 2-disposal-email | Verifies that the email address is not from a known disposable or temporary email provider before continuing. Updates the record’s status to 2-disposal-email in NocoDB for entries identified as disposable. |
| NocoDB: change status to 2-disposal-email | NocoDB                  | Mark records with disposable emails          | Check if email is Disposal (false branch) | None                             |                                                                                                              |
| Loop Over Items                   | SplitInBatches          | Process contacts one at a time              | Check if email is Disposal (true branch) | Brevo: Create Contact             | Runs one record at a time to avoid heavy load from creating contacts in bulk.                               |
| Brevo: Create Contact             | SendInBlue (Brevo)      | Create contact in Brevo                      | Loop Over Items             | NocoDB: change status to 3-contact-created | Creates a new contact in Brevo using the provided first_name, last_name, and email.                          |
| NocoDB: change status to 3-contact-created | NocoDB                  | Mark contact successfully created in Brevo  | Brevo: Create Contact       | Loop Over Items                  | Updates the record’s status to 3-contact-created in NocoDB after the contact is successfully added to Brevo. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Schedule Trigger` node:**  
   - Set interval to daily (default).  
   - This node will start the workflow once per day automatically.

3. **Add a `NocoDB` node named "NocoDB: Get new users list":**  
   - Operation: `getAll`  
   - Table: `m5mxqjs3ae68oyc` (replace with your actual table if different)  
   - Project ID: `p4lnw5vwzf2yy3i` (replace with your project ID)  
   - Options > Where: `(status,eq,0-not-imported)` to filter unprocessed contacts  
   - Authentication: Use NocoDB API Token credentials  
   - Connect output of Schedule Trigger to this node.

4. **Add an `If` node named "Check if parameters are not empty":**  
   - Version 2 conditions.  
   - Combine conditions with AND.  
   - Add conditions to check:  
     - `$json.first_name` is not empty  
     - `$json.last_name` is not empty  
     - `$json.email` is not empty  
   - Connect output of NocoDB node to this node.

5. **Add a `NocoDB` node named "NocoDB: change status to 1-empty-fields":**  
   - Operation: `update`  
   - Table: same as above  
   - Fields to update:  
     - `id` = `{{$json.Id}}` (from current item)  
     - `status` = `1-empty-fields`  
   - Authentication: same as above  
   - Connect the `false` output of the "Check if parameters are not empty" node to this node.

6. **Add an `If` node named "Check if email is Disposal":**  
   - Version 2 conditions.  
   - Condition: email does NOT match regex pattern for disposable emails.  
   - Use this regex pattern (copied exactly):  
     `.*(temp|abc|1234|yopmail|protonmail|mailinator|\.cc|bigbester|fake|spam|gdf|sdf|mr123|passinbox|landininbox|@inbox|random|anony|mymail|mail\.ru|\.buzz|asdasd|asf|simplelogin|simplelogin\.com|silomails\.com|slmails\.com|simplelogin\.fr|aleeas\.com|slmail\.me|8shield\.net|dralias\.com|passinbox\.com|passfwd\.com|passmail\.com|passmail\.net|simplelogin\.co|simplelogin\.io|duck\.com|mozmail\.com|anonaddy\.com|anonaddy\.me|trash|guerrilla|getnada|owly|mvrht|sharklasers|anonbox).*`  
   - Connect the `true` output of "Check if parameters are not empty" to this node.

7. **Add a `NocoDB` node named "NocoDB: change status to 2-disposal-email":**  
   - Operation: `update`  
   - Table: same as above  
   - Fields to update:  
     - `id` = `{{$json.Id}}`  
     - `status` = `2-disposal-email`  
   - Authentication: same as above  
   - Connect the `false` output of "Check if email is Disposal" node to this node.

8. **Add a `SplitInBatches` node named "Loop Over Items":**  
   - Default batch size of 1.  
   - Connect the `true` output of "Check if email is Disposal" to this node.

9. **Add a `SendInBlue` node named "Brevo: Create Contact":**  
   - Resource: `contact`  
   - Email: `{{$json.email}}`  
   - Attributes:  
     - FIRSTNAME = `{{$json.first_name}}`  
     - LASTNAME = `{{$json.last_name}}`  
   - Credentials: Configure Brevo API credentials (API Key)  
   - Connect output of "Loop Over Items" node (main output 2) to this node.

10. **Add a `NocoDB` node named "NocoDB: change status to 3-contact-created":**  
    - Operation: `update`  
    - Table: same as above  
    - Fields to update:  
      - `id` = `{{$json.Id}}` (from the current item in Loop Over Items node, use expression referencing that node)  
      - `status` = `3-contact-created`  
    - Authentication: same as above  
    - Connect output of "Brevo: Create Contact" to this node.

11. **Loop Back:**  
    - Connect output of "NocoDB: change status to 3-contact-created" back to the "Loop Over Items" node to continue processing the next batch item.

12. **Add sticky notes** for documentation around each logical block as described, especially for schedules, validation rules, batch processing, and status updates.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                            |
|----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| Workflow automates daily synchronization of validated contacts from NocoDB to Brevo.                                            | Workflow description                                                      |
| Regex pattern used to detect disposable email addresses includes a wide range of known disposable domains and keywords.        | Regex in "Check if email is Disposal" node                               |
| NocoDB statuses used: 0-not-imported (new), 1-empty-fields (missing data), 2-disposal-email (invalid email), 3-contact-created (imported). | Status tracking in NocoDB                                                |
| Brevo node requires API credentials; ensure API key has permissions to create contacts.                                          | Brevo API credentials                                                     |
| Batch processing set to 1 to reduce load and handle contacts sequentially.                                                      | SplitInBatches node configuration                                        |
| See n8n documentation for detailed guidance on NocoDB and SendInBlue node configurations.                                       | https://docs.n8n.io/nodes/n8n-nodes-base.nocoDb/ and https://docs.n8n.io/nodes/n8n-nodes-base.sendInBlue/ |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects the applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.