Instant Gmail Notifications for Google Form Submissions

https://n8nworkflows.xyz/workflows/instant-gmail-notifications-for-google-form-submissions-7432


# Instant Gmail Notifications for Google Form Submissions

### 1. Workflow Overview

This workflow provides **instant Gmail notifications for new Google Form submissions** by leveraging the Google Sheets integration linked to the form. Its primary use case is to alert a user via Gmail immediately after a respondent submits a Google Form, with the submission data formatted in a notification email.

The workflow is logically divided into three blocks:

- **1.1 Input Reception and Trigger:** Automatically detects new form submissions reflected as new rows in a Google Sheet.
- **1.2 Email Content Generation:** Formats the new submission data into an HTML email body.
- **1.3 Email Sending:** Sends the formatted email notification through Gmail to a designated recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

**Overview:**  
This block listens for new rows added to a Google Sheet that is linked to a Google Form. It triggers the workflow automatically every minute when a new form submission creates a new row.

**Nodes Involved:**  
- Trigger when a row is added

**Node Details:**

- **Trigger when a row is added**  
  - Type: Google Sheets Trigger  
  - Role: Workflow entry point; detects new rows in a specific Google Sheet.  
  - Configuration:  
    - Event: `rowAdded`  
    - Polling frequency: every minute  
    - Target Google Sheet: identified by URL and Sheet ID (configured as parameters)  
  - Inputs: None (trigger node)  
  - Outputs: Outputs JSON data representing the new row added to the sheet  
  - Credentials: Google Sheets OAuth2 account linked  
  - Possible Failures:  
    - Authentication errors if OAuth2 token expires or lacks permissions  
    - Network issues or Google API rate limits  
    - Incorrect sheet/document IDs causing trigger failure  
  - Notes: Requires the Google Form to be linked to the Sheet beforehand for automatic row addition.

---

#### 2.2 Email Content Generation

**Overview:**  
Formats the data from the newly added Google Sheet row into an HTML email message, extracting key fields such as the submitter’s email and their request message.

**Nodes Involved:**  
- Email generation

**Node Details:**

- **Email generation**  
  - Type: Code node (JavaScript)  
  - Role: Creates an HTML string for the email body using the new row data  
  - Configuration:  
    - JavaScript code accesses the input JSON (new row), extracting:  
      - `"Adresse e-mail"` field (if missing, defaults to "N/A")  
      - `"Please write your request"` field (if missing, defaults to "N/A")  
    - Constructs an HTML snippet with a header and the extracted data  
  - Inputs: Receives new row JSON from the Google Sheets Trigger node  
  - Outputs: JSON with an `html` property containing the email content  
  - Possible Failures:  
    - Expression errors if expected fields are missing or renamed in the sheet  
    - Code errors if input JSON structure changes  
  - Notes: This node centralizes the formatting logic, simplifying email customization.

---

#### 2.3 Email Sending

**Overview:**  
Sends the generated HTML message as an email via Gmail to a specified recipient, notifying them instantly of the new form submission.

**Nodes Involved:**  
- Instant email

**Node Details:**

- **Instant email**  
  - Type: Gmail node  
  - Role: Sends the notification email using Gmail credentials  
  - Configuration:  
    - Recipient email address set to literal string `"email"` (likely a placeholder or needs to be replaced with a real address or expression)  
    - Email subject: "The requests you received today"  
    - Message body: uses the HTML content generated in the previous node (`{{$json.html}}`)  
    - Options: Default  
  - Inputs: Receives JSON with `html` content from the Email generation node  
  - Outputs: Sends email, no further nodes connected  
  - Credentials: Gmail OAuth2 account linked  
  - Possible Failures:  
    - Authentication errors if Gmail OAuth2 token expires or lacks send permissions  
    - Incorrect recipient address causing email delivery failure  
    - Gmail sending limits or quota exceeded  
  - Notes: Requires Gmail OAuth2 credentials properly configured; the recipient address may need adjustment to a dynamic or fixed valid email.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role               | Input Node(s)                | Output Node(s)           | Sticky Note                                                                                               |
|-------------------------|----------------------------|------------------------------|-----------------------------|--------------------------|----------------------------------------------------------------------------------------------------------|
| Sticky Note1            | Sticky Note                | Information on prerequisites | None                        | None                     | ## Required\n\n- Google account\n- Google Form\n- Google Sheet linked to the form                        |
| Sticky Note              | Sticky Note                | Info on workflow trigger and data gathering | None                        | None                     | ## 1.Workflow trigger et data gathering\n\nThe workflow is **triggered automatically** as soon as a new request is received (a new row in the Google sheet from the Google Form). \n\nHow to setup:\n- Set up your Google form and link it to a google sheet\n- link the google sheet to this workflow |
| Sticky Note3            | Sticky Note                | Info on email sending step    | None                        | None                     | ## 2. Send the email\n\nThe code node, will create the message and gather all the informations wanted about the row newly added. So you'll get a new message with the new informations.\n\n\nHow to setup:\n- Connect with your Gmail account credentials. |
| Trigger when a row is added | Google Sheets Trigger      | Initiates workflow on new form submission | None                        | Email generation         |                                                                                                          |
| Email generation        | Code (JavaScript)           | Formats email HTML message    | Trigger when a row is added | Instant email            |                                                                                                          |
| Instant email           | Gmail                      | Sends notification email      | Email generation            | None                     |                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Google Sheets Trigger node:**
   - Set **Event** to `rowAdded`.
   - Configure **Document ID** by providing the URL of the Google Sheet linked to your Google Form.
   - Set **Sheet Name** to the specific sheet where form responses are recorded.
   - Set **Polling Interval** to every 1 minute.
   - Attach your **Google Sheets OAuth2 credentials** to this node.
   - Position this node as the workflow's starting point.

3. **Add a Code node named "Email generation":**
   - Connect input from the Google Sheets Trigger node.
   - Use the following JavaScript code to format the new submission into HTML:

     ```javascript
     const item = $input.item.json;

     let html = `
       <h2>📬 You just got a new question received</h2>
       <p><strong>From:</strong> ${item["Adresse e-mail"] || "N/A"}</p>
       <p><strong>Request:</strong> ${item["Please write your request"] || "N/A"}</p>
     `;

     return [{ json: { html } }];
     ```

   - This code extracts the submitter’s email and their request from the new row.

4. **Add a Gmail node named "Instant email":**
   - Connect input from the "Email generation" node.
   - Set **Send To** field to the desired recipient email address (replace `"email"` placeholder with a fixed or dynamic email).
   - Set **Subject** to `"The requests you received today"`.
   - Set **Message** to use the expression: `{{$json.html}}` to insert the HTML email content.
   - Attach your **Gmail OAuth2 credentials**.
   - Use default options unless additional Gmail settings are required (e.g., CC, BCC).

5. **Arrange nodes visually in order:**
   - Google Sheets Trigger → Email generation → Instant email.

6. **Optional: Add Sticky Notes for documentation** (recommended):  
   - Prerequisites: Google account, Google Form, linked Google Sheet  
   - Workflow trigger explanation and setup instructions  
   - Email sending instructions including Gmail credential requirements

7. **Activate the workflow** and test by submitting a new response to the linked Google Form. Confirm that the email notification is received as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                             |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| Requires a Google account with access to Google Forms, Google Sheets, and Gmail with OAuth2 credentials properly set up. | Workflow prerequisites (Sticky Note1)       |
| The Google Sheet must be linked to the Google Form to ensure new form responses create new rows, triggering the workflow. | Setup instruction (Sticky Note)              |
| Gmail node requires OAuth2 credentials with send email permission; ensure quota limits are observed.                  | Email sending info (Sticky Note3)            |

---

*Disclaimer:* This document is based exclusively on an automated workflow created with n8n, adhering strictly to current content policies. The workflow manages only legal and publicly available data without any illegal or offensive content.