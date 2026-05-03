Cost-Free Email Follow-Up Sequence with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/cost-free-email-follow-up-sequence-with-google-sheets-and-gmail-7856


# Cost-Free Email Follow-Up Sequence with Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates an email follow-up sequence using Google Sheets as a contact source and Gmail for sending emails. It is designed to run on a schedule, process a limited batch of contacts per execution, select appropriate email templates based on the contact’s progress in the sequence, send personalized emails, and update the contact status in the Google Sheet. The workflow is structured into the following logical blocks:

- **1.1 Trigger and Initialization:** Scheduling and initial settings loading.
- **1.2 Contact Retrieval and Filtering:** Fetching contacts from Google Sheets, limiting batch size, and filtering contacts who need follow-up.
- **1.3 Contact Processing Loop:** Iteratively handling each contact for validation and email sequence status.
- **1.4 Email Preparation and Sending:** Selecting email templates, formatting the email, sending via Gmail.
- **1.5 Contact Update and Loop Continuation:** Updating contact status in Google Sheets and managing wait times between batches.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

- **Overview:** This block triggers the workflow on a schedule and initializes any required settings.
- **Nodes Involved:** Schedule Trigger, Settings
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Starts the workflow automatically on a defined schedule (e.g., daily or hourly).
    - Configuration: Default scheduling parameters (not explicitly defined).
    - Inputs: None
    - Outputs: Connects to Settings node.
    - Potential Failures: Scheduling misconfiguration, system downtime.

  - **Settings**
    - Type: Set
    - Role: Initializes or sets any necessary parameters or variables before fetching contacts.
    - Configuration: Empty parameter set, likely placeholder for future configurations.
    - Inputs: Schedule Trigger
    - Outputs: Connects to Get Contacts node.
    - Potential Failures: None expected due to empty configuration.

#### 1.2 Contact Retrieval and Filtering

- **Overview:** Retrieves contacts from Google Sheets, limits the number of contacts processed, and filters out contacts already processed or who have completed the sequence.
- **Nodes Involved:** Get Contacts, Limits the number of emails per run, Filter passes only unprocessed contacts, Has contact completed email sequence?
- **Node Details:**

  - **Get Contacts**
    - Type: Google Sheets
    - Role: Reads contact data from a Google Sheet.
    - Configuration: Uses configured credentials to access a specific Google Sheet (details abstracted).
    - Inputs: Settings
    - Outputs: Limits the number of emails per run
    - Failure Types: Authentication errors, API quota limits, sheet not found, malformed data.

  - **Limits the number of emails per run**
    - Type: Limit
    - Role: Restricts the number of contacts processed in a single workflow execution to prevent overloading or hitting email limits.
    - Configuration: Default or custom limit value (not specified).
    - Inputs: Get Contacts
    - Outputs: Filter passes only unprocessed contacts
    - Failure Types: Misconfiguration leading to zero or excessive contacts.

  - **Filter passes only unprocessed contacts**
    - Type: Filter
    - Role: Filters contacts to pass only those who have not yet been processed (e.g., no prior email sent).
    - Configuration: Expression or condition checking contact status fields.
    - Inputs: Limits the number of emails per run
    - Outputs: Has contact completed email sequence?
    - Failure Types: Incorrect filtering logic causing valid contacts to be skipped.

  - **Has contact completed email sequence?**
    - Type: Filter
    - Role: Filters out contacts who have already completed the entire email follow-up sequence.
    - Configuration: Condition based on contact’s sequence status.
    - Inputs: Filter passes only unprocessed contacts
    - Outputs: Loop Over Items
    - Failure Types: Misinterpretation of sequence completion status.

#### 1.3 Contact Processing Loop

- **Overview:** Processes contacts one by one, validating their data and determining the next email step.
- **Nodes Involved:** Loop Over Items, Is Contact Real?, Determine Email Number
- **Node Details:**

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Processes contacts individually or in small batches for granular handling.
    - Configuration: Batch size not explicitly stated.
    - Inputs: Has contact completed email sequence?
    - Outputs: Wait (after processing batch), and in second output branch to Is Contact Real?
    - Failure Types: Batch size misconfiguration, loss of data integrity between batches.

  - **Is Contact Real?**
    - Type: If
    - Role: Checks if the contact data is valid and usable (e.g., valid email address).
    - Configuration: Condition to validate contact presence or data validity.
    - Inputs: Loop Over Items (second output)
    - Outputs: Determine Email Number (if true)
    - Failure Types: Incorrect validation logic leading to false negatives or positives.

  - **Determine Email Number**
    - Type: Set
    - Role: Sets a parameter to identify the next email template number based on contact progress.
    - Configuration: Likely sets a variable like `email_number` or similar.
    - Inputs: Is Contact Real?
    - Outputs: email template selector
    - Failure Types: Misassignment causing wrong template selection.

#### 1.4 Email Preparation and Sending

- **Overview:** Selects the correct email template from Google Docs, formats it as HTML, and sends the email via Gmail.
- **Nodes Involved:** email template selector, HTML: formats and beautifies the email, Send a message
- **Node Details:**

  - **email template selector**
    - Type: Google Docs
    - Role: Retrieves the email template content from Google Docs based on the email number.
    - Configuration: Uses expressions to select the correct document or template section.
    - Inputs: Determine Email Number
    - Outputs: HTML: formats and beautifies the email
    - Failure Types: Document access issues, template not found, incorrect parameterization.

  - **HTML: formats and beautifies the email**
    - Type: HTML
    - Role: Processes and beautifies raw text or template content into well-formed HTML email.
    - Configuration: May include formatting rules or style inserts.
    - Inputs: email template selector
    - Outputs: Send a message
    - Failure Types: Malformed input content, script errors.

  - **Send a message**
    - Type: Gmail
    - Role: Sends the prepared email to the contact’s email address.
    - Configuration: Uses OAuth2 credentials for Gmail, sets recipient, subject, and body.
    - Inputs: HTML: formats and beautifies the email
    - Outputs: updatecontacts
    - Failure Types: Authentication failures, Gmail API quota exceeded, invalid email addresses.

#### 1.5 Contact Update and Loop Continuation

- **Overview:** Updates the contact’s record in Google Sheets to reflect the email sent and manages wait periods between batches.
- **Nodes Involved:** updatecontacts, Wait
- **Node Details:**

  - **updatecontacts**
    - Type: Google Sheets
    - Role: Updates contact row in Google Sheets, marking email sent date/time and sequence progress.
    - Configuration: Uses `row_number` from contact data to update the correct row; sets current timestamp.
    - Inputs: Send a message
    - Outputs: Wait
    - Failure Types: Row mismatch, concurrency issues, write permission errors.

  - **Wait**
    - Type: Wait
    - Role: Pauses the workflow for a specified duration between batches to avoid email sending limits or throttling.
    - Configuration: Default or configured wait time.
    - Inputs: updatecontacts
    - Outputs: Loop Over Items (to process next batch)
    - Failure Types: Incorrect wait duration, workflow timeout.

---

### 3. Summary Table

| Node Name                           | Node Type          | Functional Role                            | Input Node(s)                      | Output Node(s)                         | Sticky Note                               |
|-----------------------------------|--------------------|--------------------------------------------|----------------------------------|---------------------------------------|-------------------------------------------|
| Schedule Trigger                  | Schedule Trigger   | Triggers workflow on schedule              | None                             | Settings                              |                                           |
| Settings                         | Set                | Initialize variables/settings               | Schedule Trigger                 | Get Contacts                         |                                           |
| Get Contacts                     | Google Sheets      | Retrieve contacts from Google Sheets        | Settings                        | Limits the number of emails per run  |                                           |
| Limits the number of emails per run | Limit              | Limit contacts processed per execution      | Get Contacts                   | Filter passes only unprocessed contacts |                                           |
| Filter passes only unprocessed contacts | Filter             | Filter contacts not yet processed           | Limits the number of emails per run | Has contact completed email sequence? |                                           |
| Has contact completed email sequence? | Filter             | Filter contacts who completed email sequence | Filter passes only unprocessed contacts | Loop Over Items                      |                                           |
| Loop Over Items                  | SplitInBatches     | Process contacts in batches/individually   | Has contact completed email sequence? | Wait (main), Is Contact Real? (secondary) |                                           |
| Is Contact Real?                 | If                 | Validate contact data                       | Loop Over Items (secondary)     | Determine Email Number                |                                           |
| Determine Email Number           | Set                | Set next email template number              | Is Contact Real?                | email template selector               |                                           |
| email template selector          | Google Docs        | Select appropriate email template            | Determine Email Number          | HTML: formats and beautifies the email |                                           |
| HTML: formats and beautifies the email | HTML               | Format email content as HTML                 | email template selector         | Send a message                       |                                           |
| Send a message                  | Gmail              | Send email via Gmail                         | HTML: formats and beautifies the email | updatecontacts                    |                                           |
| updatecontacts                 | Google Sheets      | Update contact row with email sent info     | Send a message                 | Wait                                | Row number matching used for update; timestamp added |
| Wait                            | Wait               | Pause workflow between batches               | updatecontacts                 | Loop Over Items                     |                                           |
| Sticky Note14                   | Sticky Note        | (No content)                                | None                          | None                                |                                           |
| Sticky Note15                   | Sticky Note        | (No content)                                | None                          | None                                |                                           |
| Sticky Note1                    | Sticky Note        | (No content)                                | None                          | None                                |                                           |
| Sticky Note                     | Sticky Note        | (No content)                                | None                          | None                                |                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Purpose: Automatically starts workflow on schedule (e.g., daily at a set time).  
   - Configure timing as needed.

2. **Add a Set node named “Settings”**  
   - Type: Set  
   - Purpose: Placeholder to initialize variables or settings (empty parameters).  
   - Connect Schedule Trigger → Settings.

3. **Add Google Sheets node “Get Contacts”**  
   - Type: Google Sheets (Read)  
   - Configure credentials for Google Sheets.  
   - Select the spreadsheet and worksheet containing contacts.  
   - Connect Settings → Get Contacts.

4. **Add Limit node “Limits the number of emails per run”**  
   - Type: Limit  
   - Configure maximum number of contacts to process per workflow run (e.g., 10).  
   - Connect Get Contacts → Limit node.

5. **Add Filter node “Filter passes only unprocessed contacts”**  
   - Type: Filter  
   - Configure condition to allow only contacts where “processed” or “email sent” flag is false or empty.  
   - Connect Limit → Filter.

6. **Add Filter node “Has contact completed email sequence?”**  
   - Type: Filter  
   - Configure condition to exclude contacts who have finished the full sequence (e.g., sequence step equals max).  
   - Connect Filter passes only unprocessed contacts → Has contact completed email sequence?.

7. **Add SplitInBatches node “Loop Over Items”**  
   - Type: SplitInBatches  
   - Configure batch size 1 (or desired batch size).  
   - Connect Has contact completed email sequence? → Loop Over Items.

8. **Add Wait node “Wait”**  
   - Type: Wait  
   - Configure wait time between batches to avoid rate limits (e.g., 1 minute).  
   - Connect Loop Over Items (main output) → Wait → Loop Over Items (to continue loop).

9. **Add If node “Is Contact Real?”**  
   - Type: If  
   - Condition: Check if contact email exists and is valid (e.g., non-empty string, regex for email).  
   - Connect Loop Over Items (second output) → Is Contact Real? (true branch).

10. **Add Set node “Determine Email Number”**  
    - Type: Set  
    - Configure to set variable representing the next email number, based on contact’s current sequence step.  
    - Connect Is Contact Real? (true) → Determine Email Number.

11. **Add Google Docs node “email template selector”**  
    - Type: Google Docs  
    - Configure credentials and specify dynamic selection of document/template based on email number variable.  
    - Connect Determine Email Number → email template selector.

12. **Add HTML node “HTML: formats and beautifies the email”**  
    - Type: HTML  
    - Configure to process raw template content into HTML format, adding styling as needed.  
    - Connect email template selector → HTML formatting.

13. **Add Gmail node “Send a message”**  
    - Type: Gmail  
    - Configure OAuth2 credentials for Gmail account.  
    - Set recipient email as contact email, subject and body as formatted HTML content.  
    - Connect HTML formatting → Send a message.

14. **Add Google Sheets node “updatecontacts”**  
    - Type: Google Sheets (Update)  
    - Configure to update the contact row using `row_number` to mark email sent date/time and increment sequence step.  
    - Connect Send a message → updatecontacts.

15. **Connect updatecontacts → Wait**  
    - This completes the batch cycle and allows for pause before next batch.

16. **Add Sticky Notes as desired for documentation or reminders at relevant points.**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                      |
|------------------------------------------------------------------------------------------------------------------|------------------------------------|
| Row number is used for matching contact rows during update to ensure correct record is modified.                 | updatecontacts node note            |
| Timestamp format for processing is `yyyy-MM-dd, hh:mm:ss a` for consistent date-time logging.                   | updatecontacts node note            |
| Workflow respects Gmail sending limits by batching and waiting between batches to avoid throttling or blocks.   | Wait node purpose                   |
| Use Google Docs for dynamic email templates allows easy template management outside n8n.                        | email template selector node usage |
| For Gmail OAuth2 credentials setup, ensure proper scopes are granted for sending emails and accessing user data. | Gmail node credential requirement  |

---

**Disclaimer:** The provided description and analysis are based exclusively on the automated workflow created using n8n, respecting all content policies. All handled data is legal and public.