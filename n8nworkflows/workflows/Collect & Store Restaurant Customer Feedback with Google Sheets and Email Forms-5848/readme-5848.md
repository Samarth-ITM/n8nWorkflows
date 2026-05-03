Collect & Store Restaurant Customer Feedback with Google Sheets and Email Forms

https://n8nworkflows.xyz/workflows/collect---store-restaurant-customer-feedback-with-google-sheets-and-email-forms-5848


# Collect & Store Restaurant Customer Feedback with Google Sheets and Email Forms

### 1. Workflow Overview

This workflow is designed for collecting and managing customer feedback related to food and restaurant experiences. It facilitates feedback submission via an online form, stores the collected data in a Google Sheet, and optionally notifies a team member by email when new feedback is received. The workflow is structured into the following logical blocks:

- **1.1 Feedback Submission Reception:** Captures feedback submitted through an embedded form.
- **1.2 Data Processing and Storage:** Waits briefly before appending or updating the submitted feedback into a Google Sheet.
- **1.3 Feedback Entry Monitoring and Notification:** Watches the Google Sheet for new entries and triggers an email notification to alert the team.
- **1.4 Auxiliary Wait and Logging:** Includes wait and code nodes to manage timing and simple logging for process flow control.

---

### 2. Block-by-Block Analysis

#### 2.1 Feedback Submission Reception

- **Overview:**  
  This block captures customer feedback submitted via a web form hosted on n8n's form interface. It receives form data, initiates a short wait to ensure completeness, then proceeds to store the data.

- **Nodes Involved:**  
  - Trigger: Form Submitted  
  - Wait: Pause Before Processing  
  - Append or Update Row in Google Sheet  

- **Node Details:**

  - **Trigger: Form Submitted**  
    - Type: Form Trigger (webhook-based)  
    - Configuration: Listens on path `/feedback/list` with custom CSS to style the form, including branding and input styles. The form collects fields such as Name, Email, Contact Number, cleanliness rating, food taste, dish enjoyed, order accuracy, staff politeness, food presentation, overall dining rating, and additional comments. All fields are required.  
    - Key Expressions: Uses form field labels as keys in JSON output, e.g., `$json['Email ID']`.  
    - Input: HTTP POST from form submission  
    - Output: JSON object of form responses with submission timestamp (`submittedAt`).  
    - Failure Cases: Webhook failure, invalid or incomplete form data, CSS loading issues.  
    - Notes: The form is customized with a branded logo and clean input styling.  
   
  - **Wait: Pause Before Processing**  
    - Type: Wait node  
    - Configuration: Pauses workflow execution for 10 seconds after receiving form submission.  
    - Purpose: Ensures any asynchronous operations or data availability stabilize before continuing.  
    - Input: Trigger: Form Submitted output  
    - Output: Passes data unchanged after wait  
    - Failure Cases: Rare, possible timeout or workflow stop during wait.
   
  - **Append or Update Row in Google Sheet**  
    - Type: Google Sheets node (appendOrUpdate operation)  
    - Configuration: Writes form data into a Google Sheet named "Food_feedback" (sheet ID 2016314922) within a specified spreadsheet. It matches existing rows by "E-Mail" column to update or appends new rows if none matched. All form fields are mapped explicitly to columns, including a timestamp from `submittedAt`.  
    - Key Expressions: Uses expressions like `{{$json['What is your Name?']}}` to map data to sheet columns.  
    - Input: Output from Wait node  
    - Output: Operation result of update/append  
    - Failure Cases: Credential issues, API rate limits, schema mismatches, missing columns, or incorrect data types.  
    - Credential: Google Sheets OAuth2 account configured.  

#### 2.2 Feedback Entry Monitoring and Notification

- **Overview:**  
  This block monitors the Google Sheet for newly added feedback entries and sends an email notification to the respective user or team informing them about the feedback request or confirmation.

- **Nodes Involved:**  
  - Google Sheets Trigger: New Feedback Entry  
  - Wait for All Data (Code)  
  - Send Email: Notify Team About Feedback  

- **Node Details:**

  - **Google Sheets Trigger: New Feedback Entry**  
    - Type: Google Sheets Trigger node  
    - Configuration: Polls every minute for changes in the sheet with `gid=0` (likely a summary or user detail sheet) in the same spreadsheet used for feedback. Triggers when new rows appear.  
    - Input: Polling trigger  
    - Output: New rows detected in the sheet as JSON objects  
    - Failure Cases: Credential expiration, API limits, connectivity issues.  
    - Credential: Google Sheets Trigger OAuth2 account used.  

  - **Wait for All Data (Code)**  
    - Type: Code node (JavaScript)  
    - Configuration: Includes a console log and a setTimeout for 5 seconds (non-blocking in this version, effectively immediate continuation)  
    - Purpose: Acts as a placeholder for delay or logging; might be intended for flow control or debugging.  
    - Input: Output from Google Sheets Trigger  
    - Output: Passes data unchanged  
    - Failure Cases: Runtime errors in code node (unlikely here).  

  - **Send Email: Notify Team About Feedback**  
    - Type: Email Send node  
    - Configuration: Sends a plain text email with subject "Feedback" to the email address extracted from the sheet entry (`{{$json.Email}}`). The email body requests feedback with a link to the form. From email is set as "vrushti@itoneclick.com".  
    - Input: Output from code node  
    - Output: Email sending status  
    - Failure Cases: SMTP authentication failure, invalid email addresses, network issues.  
    - Credential: SMTP account configured for sending email.  

#### 2.3 Auxiliary Wait and Logging

- **Overview:**  
  Implements a small coded delay and console logs primarily for monitoring or debugging purposes in the workflow's feedback notification path.

- **Nodes Involved:**  
  - Wait for All Data (Code)  

- **Node Details:**  
  - Covered above in Section 2.2.

---

### 3. Summary Table

| Node Name                          | Node Type                | Functional Role                          | Input Node(s)              | Output Node(s)                        | Sticky Note                                                                                 |
|-----------------------------------|--------------------------|----------------------------------------|----------------------------|-------------------------------------|---------------------------------------------------------------------------------------------|
| Google Sheets Trigger: New Feedback Entry | Google Sheets Trigger     | Monitor new feedback entries in sheet   |                            | Wait for All Data (Code)             |                                                                                             |
| Wait for All Data (Code)           | Code                     | Delay and logging before sending email  | Google Sheets Trigger       | Send Email: Notify Team About Feedback |                                                                                             |
| Send Email: Notify Team About Feedback | Email Send               | Notify team/user about feedback via email | Wait for All Data (Code)    |                                     |                                                                                             |
| Trigger: Form Submitted            | Form Trigger             | Capture customer feedback form submissions |                            | Wait: Pause Before Processing       |                                                                                             |
| Wait: Pause Before Processing      | Wait                     | Pause to ensure data completeness       | Trigger: Form Submitted     | Append or Update Row in Google Sheet |                                                                                             |
| Append or Update Row in Google Sheet | Google Sheets            | Store or update feedback data in sheet  | Wait: Pause Before Processing |                                     |                                                                                             |
| Sticky Note                       | Sticky Note              | Workflow description                     |                            |                                     | ## Description\n\nThis workflow collects customer feedback about food and restaurant experience through a form, stores the responses in a Google Sheet, and optionally notifies the team via email. It helps track guest satisfaction and improve service quality. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a **Form Trigger** node named "Trigger: Form Submitted".  
   - Configure the webhook path as `feedback/list`.  
   - Insert the custom CSS provided to style the form (including branding and input styles).  
   - Define form fields exactly as:  
     - "What is your Name?" (text, required)  
     - "Email ID" (email, required)  
     - "Contact Number" (text, required, placeholder: "(e.g, +1234567890)")  
     - "How was the cleanliness of the dining area?" (dropdown with 5 satisfaction options, required)  
     - "Did you like the taste of the food?" (dropdown Yes/No, required)  
     - "What dish did you enjoy the most?" (text, required)  
     - "Was your order accurate and timely?" (dropdown Yes/No, required)  
     - "Was our staff polite and helpful?" (dropdown Yes/No, required)  
     - "Was the food presentation appealing?" (dropdown with Good, Average, Bad, Very Bad, required)  
     - "How would you rate your overall dining experience?" (dropdown 1 to 5, required)  
     - "Any additional comments or suggestions?" (textarea, required)  
   - Enable submission timestamp capture.

2. **Add a Wait Node (Pause Before Processing)**  
   - Insert a **Wait** node named "Wait: Pause Before Processing".  
   - Set wait time to 10 seconds.  
   - Connect output of "Trigger: Form Submitted" to this node.

3. **Add Google Sheets Node to Append or Update Data**  
   - Add a **Google Sheets** node named "Append or Update Row in Google Sheet".  
   - Set operation to `appendOrUpdate`.  
   - Configure Google Sheets OAuth2 credentials.  
   - Select the spreadsheet by its ID (e.g., `1WFcRiJk_kh_1UZo9PzS07Fsl5WS7zviePzhFOz9hFaA`).  
   - Choose the sheet tab "Food_feedback" by sheet ID `2016314922`.  
   - Define column mapping with explicit expressions for each form field, including a "Timestamp" column mapped to the submission time (`{{$json.submittedAt}}`).  
   - Use "E-Mail" as matching column to update existing rows or append new ones.  
   - Connect output of Wait node to this Google Sheets node.

4. **Set Up Google Sheets Trigger Node**  
   - Add a **Google Sheets Trigger** node named "Google Sheets Trigger: New Feedback Entry".  
   - Configure it to poll every minute on the spreadsheet `1WFcRiJk_kh_1UZo9PzS07Fsl5WS7zviePzhFOz9hFaA`, sheet `gid=0`.  
   - Use Google Sheets Trigger OAuth2 credentials.  
   - This node will activate when new rows are added or updated.

5. **Add Code Node for Delay / Logging**  
   - Add a **Code** node named "Wait for All Data (Code)".  
   - Insert the JavaScript code to log "Waiting..." and schedule a 5-second timeout (note: non-blocking, serves mainly as a placeholder).  
   - Connect output of Google Sheets Trigger node to this Code node.

6. **Add Email Send Node**  
   - Add an **Email Send** node named "Send Email: Notify Team About Feedback".  
   - Use SMTP credentials configured for sending emails.  
   - Set the email `To` field as `={{ $json.Email }}` from sheet data.  
   - Set the `From` field as `vrushti@itoneclick.com`.  
   - Email subject: "Feedback".  
   - Email body:  
     ```
     Please give us your valuable feedback at:
     https://n8n-devops.oneclicksales.xyz/form/feedback/list
     ```  
   - Connect output of Code node to this Email Send node.

7. **Add Sticky Note**  
   - Add a **Sticky Note** node with the description:  
     ```
     ## Description

     This workflow collects customer feedback about food and restaurant experience through a form, stores the responses in a Google Sheet, and optionally notifies the team via email. It helps track guest satisfaction and improve service quality.
     ```

8. **Connect Nodes According to the Workflow:**  
   - "Trigger: Form Submitted" → "Wait: Pause Before Processing" → "Append or Update Row in Google Sheet"  
   - "Google Sheets Trigger: New Feedback Entry" → "Wait for All Data (Code)" → "Send Email: Notify Team About Feedback"

9. **Activate the Workflow**  
   - Ensure all credentials are properly configured and authorized.  
   - Activate the workflow to start listening for form submissions and sheet changes.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Custom form styling includes a company logo and modern, accessible input styles via embedded CSS.                   | CSS customization within Form Trigger node parameters        |
| Feedback form URL used in email notification: https://n8n-devops.oneclicksales.xyz/form/feedback/list                | Email Send node body                                         |
| Google Sheets document used for storage: https://docs.google.com/spreadsheets/d/1WFcRiJk_kh_1UZo9PzS07Fsl5WS7zviePzhFOz9hFaA | Referenced in Google Sheets nodes                            |
| SMTP account used for email sending is identified as "SMTP account" credential within n8n.                           | Email Send node credentials                                   |

---

This documentation enables a full understanding of the workflow’s structure, data flow, node functions, configuration details, and how to rebuild or modify it effectively. It also anticipates potential error cases such as credential failures, API limits, and form submission issues.