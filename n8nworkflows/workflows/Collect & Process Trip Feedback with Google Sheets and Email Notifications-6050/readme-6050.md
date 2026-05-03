Collect & Process Trip Feedback with Google Sheets and Email Notifications

https://n8nworkflows.xyz/workflows/collect---process-trip-feedback-with-google-sheets-and-email-notifications-6050


# Collect & Process Trip Feedback with Google Sheets and Email Notifications

### 1. Workflow Overview

This workflow automates the collection and processing of trip feedback using a Google Sheets form submission and email notifications. It targets organizations or travel companies that want to streamline gathering customer feedback, storing it in a structured Google Sheet, and promptly sending follow-up emails to respondents.

The workflow is organized into the following logical blocks:

- **1.1 Trigger Input Reception:** Detects new trip feedback form submissions and new Google Sheets entries.
- **1.2 Processing Delay:** Introduces a wait period to ensure data consistency before further processing.
- **1.3 Feedback Data Iteration:** Splits multiple feedback items for individual processing.
- **1.4 Data Storage:** Appends or updates feedback data in a centralized Google Sheets document.
- **1.5 Notification:** Sends an email containing the feedback form link to newly registered users.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Input Reception

**Overview:**  
This block captures new user feedback submissions either via a custom form or directly from rows added in a Google Sheets document, initiating the workflow process.

**Nodes Involved:**  
- Trigger - Trip Form Submission  
- Trigger - New User Entry  

**Node Details:**

- **Trigger - Trip Form Submission**  
  - Type: Form Trigger  
  - Role: Starts workflow when a user submits the trip feedback form hosted at `/form/trip-feedback`.  
  - Configuration:  
    - Custom CSS applied for consistent branding and styling, including a company logo and clean input field design.  
    - Form fields include Name, Email, Contact Number, dropdowns for satisfaction and experience ratings, and a textarea for comments.  
    - All fields are required, ensuring complete data capture.  
  - Inputs: None (trigger)  
  - Outputs: Passes form submission data downstream.  
  - Edge Cases: Form submission failure, incomplete data if client-side validation bypassed, network issues.  
  - Version: 2.2  

- **Trigger - New User Entry**  
  - Type: Google Sheets Trigger  
  - Role: Starts the workflow when a new row is added to a specified Google Sheets document (ID and sheet URL partially obfuscated).  
  - Configuration:  
    - Polling every hour at the 10th minute to check for new rows.  
    - OAuth2 credentials used for Google Sheets access.  
  - Inputs: None (trigger)  
  - Outputs: Passes new row data downstream.  
  - Edge Cases: Google API rate limits, credentials expiry, sheet renaming or ID changes causing failures.  
  - Version: 1  

---

#### 1.2 Processing Delay

**Overview:**  
Introduces a fixed delay to ensure that the data received from Google Sheets trigger is fully processed and stable before further workflow steps are executed.

**Nodes Involved:**  
- Delay - Process Buffer  

**Node Details:**

- **Delay - Process Buffer**  
  - Type: Wait node  
  - Role: Delays the workflow execution by 7 seconds.  
  - Configuration:  
    - Fixed 7 seconds delay to buffer the process.  
  - Inputs: From "Trigger - New User Entry"  
  - Outputs: To "Send Email To That New User"  
  - Edge Cases: Workflow timeout if delay is too long, or premature downstream execution if delay is too short.  
  - Version: 1.1  

---

#### 1.3 Feedback Data Iteration

**Overview:**  
Processes each feedback submission item individually, allowing batch processing of multiple entries if submitted at once.

**Nodes Involved:**  
- Tack All Feedback Item  

**Node Details:**

- **Tack All Feedback Item**  
  - Type: SplitInBatches  
  - Role: Iterates over each item in the incoming data set to process them sequentially or in manageable batches.  
  - Configuration: Default batch settings (no explicit batch size specified).  
  - Inputs: From "Trigger - Trip Form Submission"  
  - Outputs: To "Update - Trip Feedback Sheet" and itself (for iterative continuation).  
  - Edge Cases: Large batch size causing memory issues, empty batches causing no processing.  
  - Version: 3  

---

#### 1.4 Data Storage

**Overview:**  
Appends or updates trip feedback data into a Google Sheets spreadsheet to maintain a centralized, organized record of all feedback responses.

**Nodes Involved:**  
- Update - Trip Feedback Sheet  

**Node Details:**

- **Update - Trip Feedback Sheet**  
  - Type: Google Sheets  
  - Role: Inserts or updates rows in the "form" sheet of a specific Google Sheets document (ID partially obfuscated).  
  - Configuration:  
    - Operation: appendOrUpdate to add new records or update existing ones.  
    - Mapping mode: Automatically maps input data fields to sheet columns.  
    - Authentication via service account with Google API credentials.  
  - Inputs: From "Tack All Feedback Item" (per batch item)  
  - Outputs: None (terminal node for this branch)  
  - Edge Cases: Permission errors on the Google Sheet, API quota limits, schema mismatch between input data and sheet columns.  
  - Version: 4.5  

---

#### 1.5 Notification

**Overview:**  
Sends an email invitation to the new user to complete the trip feedback form, providing a direct link and ensuring prompt engagement.

**Nodes Involved:**  
- Send Email To That New User  

**Node Details:**

- **Send Email To That New User**  
  - Type: Email Send  
  - Role: Sends a plain text email with a link to the feedback form to the email address provided in the submission.  
  - Configuration:  
    - Subject: "Feedback"  
    - From: abc@gmail.com (placeholder email, should be replaced with valid sender)  
    - To: Dynamically set to `{{$json.Email}}` (email from form submission)  
    - Email format: Plain text  
    - SMTP credentials configured for sending emails.  
  - Inputs: From "Delay - Process Buffer"  
  - Outputs: None (terminal node for this branch)  
  - Edge Cases: Email sending failure due to SMTP errors, invalid email addresses, spam filtering issues.  
  - Version: 2.1  

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                                     | Input Node(s)             | Output Node(s)                    | Sticky Note                                                                                                         |
|---------------------------|------------------------|----------------------------------------------------|---------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Trigger - Trip Form Submission | Form Trigger           | Triggers workflow on trip feedback form submission | None                      | Tack All Feedback Item           | This node triggers the workflow when a trip feedback form is submitted, initiating the data processing loop.         |
| Tack All Feedback Item     | SplitInBatches         | Iterates over each submission item for processing  | Trigger - Trip Form Submission | Update - Trip Feedback Sheet, Tack All Feedback Item | This node iterates over each form submission item to process multiple entries if present, ensuring all data is handled. |
| Update - Trip Feedback Sheet | Google Sheets          | Appends/updates feedback data in Google Sheets     | Tack All Feedback Item     | None                            | This node appends or updates the trip feedback data in the Google Sheets, maintaining an organized record.           |
| Trigger - New User Entry   | Google Sheets Trigger  | Triggers workflow on new Google Sheets row addition | None                      | Delay - Process Buffer           | This node triggers the workflow whenever a new row is added to the Google Sheets feedback form.                      |
| Delay - Process Buffer     | Wait                   | Adds delay to ensure data is processed before next steps | Trigger - New User Entry   | Send Email To That New User      | This node introduces a delay to ensure the data is fully processed before sending notifications, avoiding premature actions. |
| Send Email To That New User | Email Send             | Sends feedback form email to newly registered users | Delay - Process Buffer     | None                            | This node sends an email with feedback form to the new user.                                                         |
| Sticky Note               | Sticky Note            | Informational comment                              | None                      | None                            | This node triggers the workflow whenever a new row is added to the Google Sheets feedback form.                      |
| Sticky Note1              | Sticky Note            | Informational comment                              | None                      | None                            | This node introduces a delay to ensure the data is fully processed before sending notifications, avoiding premature actions. |
| Sticky Note2              | Sticky Note            | Informational comment                              | None                      | None                            | This node sends an email with feedback form to the new user.                                                         |
| Sticky Note3              | Sticky Note            | Informational comment                              | None                      | None                            | This node appends or updates the trip feedback data in the Google Sheets, maintaining an organized record.           |
| Sticky Note4              | Sticky Note            | Informational comment                              | None                      | None                            | This node iterates over each form submission item to process multiple entries if present, ensuring all data is handled. |
| Sticky Note5              | Sticky Note            | Informational comment                              | None                      | None                            | This node triggers the workflow when a trip feedback form is submitted, initiating the data processing loop.         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger - Trip Form Submission**  
   - Add a **Form Trigger** node.  
   - Set webhook path to `trip-feedback`.  
   - Configure form with:  
     - Fields: Name (text), Email (email), Contact Number (text), multiple dropdowns for satisfaction, food taste, tour guide politeness, overall rating, and a textarea for comments.  
     - All fields marked required.  
   - Apply custom CSS for branding and styling as per the current configuration or your design.  
   - Save and activate webhook.

2. **Create Tack All Feedback Item**  
   - Add a **SplitInBatches** node named "Tack All Feedback Item".  
   - Connect output of "Trigger - Trip Form Submission" to this node.  
   - Leave batch size default, or configure if large datasets expected.

3. **Create Update - Trip Feedback Sheet**  
   - Add a **Google Sheets** node named "Update - Trip Feedback Sheet".  
   - Set operation to `appendOrUpdate`.  
   - Set Google Sheet document ID (replace with your own): `"9iuygtfr56yuhjn"`.  
   - Set sheet name to `"form"`.  
   - Set authentication to use a **Google Service Account** credential with proper access rights.  
   - Map incoming data automatically to sheet columns.  
   - Connect output of "Tack All Feedback Item" to this node.

4. **Create Trigger - New User Entry**  
   - Add a **Google Sheets Trigger** node named "Trigger - New User Entry".  
   - Configure with your Google Sheet document ID and sheet name (match your feedback sheet).  
   - Set event to trigger on "rowAdded".  
   - Set polling interval to every hour at minute 10 or adjust as needed.  
   - Use OAuth2 credentials for Google Sheets access.

5. **Create Delay - Process Buffer**  
   - Add a **Wait** node named "Delay - Process Buffer".  
   - Set fixed wait time to 7 seconds.  
   - Connect output of "Trigger - New User Entry" to this node.

6. **Create Send Email To That New User**  
   - Add an **Email Send** node named "Send Email To That New User".  
   - Set subject to `"Feedback"`.  
   - Set from email to your valid sender email address.  
   - Set to email dynamically to `{{$json.Email}}` (email from trigger data).  
   - Use plain text format.  
   - Insert email body text:  
     ```
     Please fill out this feedback form:
     https://n8n-devops.oneclicksales.xyz/form/trip-feedback
     ```  
   - Configure SMTP credentials for your email provider.  
   - Connect output of "Delay - Process Buffer" to this node.

7. **Connect Workflow**  
   - Connect "Trigger - Trip Form Submission" → "Tack All Feedback Item" → "Update - Trip Feedback Sheet".  
   - Connect "Trigger - New User Entry" → "Delay - Process Buffer" → "Send Email To That New User".

8. **Add Sticky Notes (Optional)**  
   - Add sticky notes near corresponding nodes describing their function for maintenance and clarity.

9. **Activate Workflow**  
   - Enable the workflow and test by submitting the form and adding rows to the Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The form includes embedded company logo via CSS background-image for branding purposes.           | CSS URL: https://d1rdz15x9x7c4f.cloudfront.net/assets/payload-images/oc-blue-logo.svg                     |
| Custom CSS hides default n8n branding and footer elements for a professional user experience.     | Applied in Form Trigger node parameters.                                                                 |
| SMTP and Google Sheets credentials must be properly configured and tested for successful execution. | Ensure OAuth2 tokens and service accounts have correct scopes and permissions.                            |
| Workflow delay ensures data consistency before sending emails, preventing premature notifications. | Avoid reducing wait time below 7 seconds unless confirmed reliable in your environment.                   |
| The Google Sheets node uses appendOrUpdate to handle both new and existing feedback records.      | Mapping mode is auto; verify your sheet column headers match form data keys.                              |
| Google Sheets trigger polling interval is hourly; adjust for near real-time triggers if needed.   | Frequent polling may affect API quota and performance.                                                   |

---

**Disclaimer:**  
The provided content is extracted exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.