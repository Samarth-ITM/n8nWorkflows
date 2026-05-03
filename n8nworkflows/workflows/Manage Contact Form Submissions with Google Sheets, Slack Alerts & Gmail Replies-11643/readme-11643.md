Manage Contact Form Submissions with Google Sheets, Slack Alerts & Gmail Replies

https://n8nworkflows.xyz/workflows/manage-contact-form-submissions-with-google-sheets--slack-alerts---gmail-replies-11643


# Manage Contact Form Submissions with Google Sheets, Slack Alerts & Gmail Replies

### 1. Workflow Overview

This workflow automates the processing of contact form submissions by integrating form data capture, data storage, notification, and reply mechanisms. It targets use cases where organizations want to efficiently handle inquiries by:

- Capturing user submissions from a web contact form.
- Storing submissions in a Google Sheets document as an inquiry log.
- Sending Slack notifications to alert team members with inquiry details.
- Enabling replies to inquiries directly via Slack-triggered emails.
- Confirming submission completion to the user via the form interface.

The workflow’s logic is grouped into the following functional blocks:

- **1.1 Input Reception:** Triggered by form submission and initial data structuring.
- **1.2 Data Persistence:** Append the submission data to a Google Sheets inquiry list.
- **1.3 Slack Notification:** Compose and send a formatted Slack message alert with inquiry details and action button.
- **1.4 Email Reply Handling:** Receive Slack action callbacks via a webhook, parse parameters, and send a confirmation reply email to the inquirer.
- **1.5 Form Completion:** Display a thank-you message to the form submitter after successful processing.
- **1.6 Configuration:** Centralized node to assign key variables for reuse in notifications and processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block triggers the workflow on new contact form submissions, captures and assigns form data to variables for downstream nodes.
- **Nodes Involved:**  
  - On form submission  
  - Config  
  - NewRecord

- **Node Details:**

  1. **On form submission**  
     - *Type & Role:* `formTrigger` node; listens for submissions to the "Inquiry Form".  
     - *Configuration:* Form fields include "Name" (required), "Email address" (required), and "Meesage" (textarea, required). The form has a description thanking visitors.  
     - *Expressions:* N/A  
     - *Connections:* Outputs to "Config".  
     - *Failures:* Possible webhook errors or malformed submissions.  
     - *Version:* 2.3  

  2. **Config**  
     - *Type & Role:* `set` node; centralizes assignment of variables from form data for reuse.  
     - *Configuration:* Assigns variables:  
       - `name` from form field "Name"  
       - `emailAddress` from "Email address"  
       - `message` from "Meesage"  
       - `slackMessage` builds a Slack formatted message including name, email, and message with markdown and code block formatting.  
       - `contactWebhookUrl` set to a placeholder webhook URL for Slack button action.  
     - *Expressions:* Uses expressions like `={{ $json['Name'] }}` to pull form data dynamically.  
     - *Connections:* Outputs to "NewRecord".  
     - *Failures:* Expression errors if expected fields missing.  
     - *Version:* 3.4  

  3. **NewRecord**  
     - *Type & Role:* `code` node; restructures JSON to simplify downstream usage.  
     - *Configuration:* Copies `name`, `emailAddress`, and `message` from input JSON to output JSON.  
     - *Expressions:* JavaScript code accessing `$input.first().json`.  
     - *Connections:* Outputs to "Append row in inquiry list".  
     - *Failures:* Runtime JS errors if input missing keys.  
     - *Version:* 2  

#### 1.2 Data Persistence

- **Overview:** Appends the contact submission as a new row to a Google Sheets document to maintain a persistent inquiry log.
- **Nodes Involved:**  
  - Append row in inquiry list

- **Node Details:**

  1. **Append row in inquiry list**  
     - *Type & Role:* `googleSheets` node; appends a new row to a specific Google Sheet.  
     - *Configuration:*  
       - Operation: append  
       - Sheet Name: "gid=0" (first sheet)  
       - Document ID: Google Sheet ID specified  
       - Columns mapped automatically from input data fields: "Email address", "Meesage", "submittedAt", "formMode" (submittedAt and formMode may be empty unless provided upstream).  
     - *Expressions:* Auto mapping input data.  
     - *Connections:* Outputs to "Send a message" (Slack notification).  
     - *Failures:* OAuth errors, permission issues, API rate limits.  
     - *Version:* 4.7  
     - *Credentials:* Google Sheets OAuth2  

#### 1.3 Slack Notification

- **Overview:** Sends a formatted Slack message with inquiry details, containing a "Contact" button to trigger email reply.
- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  1. **Send a message**  
     - *Type & Role:* `slack` node; sends a block-formatted message to a Slack channel.  
     - *Configuration:*  
       - Channel ID: placeholder "Please change here" (must be set to actual channel ID).  
       - Message Type: block  
       - Blocks: JSON block structure including:  
         - Section block with markdown text from `slackMessage` variable (with newline escapes).  
         - Divider block.  
         - Actions block with one button labeled "Contact" that links to the `contactWebhookUrl` with query parameters `name` and `emailAddress` appended.  
     - *Expressions:* Uses `$('Config').item.json` to access variables.  
     - *Connections:* Outputs to "Form ending".  
     - *Failures:* Slack API errors, invalid channel, broken URLs.  
     - *Version:* 2.3  
     - *Credentials:* Slack API OAuth  

#### 1.4 Email Reply Handling

- **Overview:** Handles incoming webhook requests triggered by the Slack "Contact" button, extracts parameters, and sends an acknowledgment email to the inquiry sender.
- **Nodes Involved:**  
  - ContactWebhook  
  - EmailContent  
  - Send a email

- **Node Details:**

  1. **ContactWebhook**  
     - *Type & Role:* `webhook` node; receives incoming HTTP requests with basic authentication.  
     - *Configuration:*  
       - Path: unique UUID path for webhook  
       - Authentication: Basic Auth enabled using stored credentials.  
     - *Connections:* Outputs to "EmailContent".  
     - *Failures:* Auth errors, missing parameters in query string.  
     - *Version:* 2.1  
     - *Credentials:* Basic Auth  

  2. **EmailContent**  
     - *Type & Role:* `code` node; extracts `name` and `emailAddress` from webhook query parameters, validates presence.  
     - *Configuration:*  
       - If either `name` or `emailAddress` missing, returns empty to stop processing.  
       - Otherwise, outputs JSON with `name` and `emailAddress`.  
     - *Expressions:* JavaScript code accessing `raw.query`.  
     - *Connections:* Outputs to "Send a email".  
     - *Failures:* Missing query parameters, runtime JS errors.  
     - *Version:* 2  

  3. **Send a email**  
     - *Type & Role:* `gmail` node; sends a confirmation email to the inquirer.  
     - *Configuration:*  
       - To: uses `$json.emailAddress` from previous node.  
       - Subject: fixed string "We received your message".  
       - Message: personalized text using `$json.name` in plain text format.  
     - *Connections:* None (end node).  
     - *Failures:* Gmail OAuth errors, invalid email addresses, quota limits.  
     - *Version:* 2.1  
     - *Credentials:* Gmail OAuth2  

#### 1.5 Form Completion

- **Overview:** Displays a completion message in the form interface after submission and processing.
- **Nodes Involved:**  
  - Form ending

- **Node Details:**

  1. **Form ending**  
     - *Type & Role:* `form` node; performs form completion operation.  
     - *Configuration:*  
       - Completion title: "Thank you for contacting us."  
       - Completion message: "We have received your message and will get back to you shortly."  
     - *Connections:* Receives input from "Send a message" (Slack notification success).  
     - *Failures:* Form UI errors, webhook delivery issues.  
     - *Version:* 2.3  

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                         | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                                     |
|-------------------------|-------------------|---------------------------------------|------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| On form submission      | formTrigger       | Trigger on form submission             | —                      | Config                    | ## How it works This workflow is triggered when the contact form is submitted. It automatically saves the inquiry details to Google Sheets and sends a notification to Slack. You can then review the inquiry and reply directly from Slack using the `Contact` button.  |
| Config                 | set               | Assign variables from form data        | On form submission     | NewRecord                 | * How to use * Open the `Gmail` node and set up the Credential. * Open the `Google Sheets` node and set up the Credential. * Open the `Slack` node and set up the Credential to allow sending messages. You can create a new Slack App [here](https://api.slack.com/apps). * Open the `ContactWebhook` node and configure Basic Auth. * Open the `Config` node and update the `contactWebhookUrl` parameter to match the Production URL from the `ContactWebhook` node. |
| NewRecord              | code              | Restructure JSON for downstream nodes | Config                 | Append row in inquiry list |                                                                                                                                 |
| Append row in inquiry list | googleSheets      | Append submission data to Google Sheets | NewRecord              | Send a message            |                                                                                                                                 |
| Send a message         | slack             | Send Slack notification with inquiry  | Append row in inquiry list | Form ending             | * Customizing this workflow * You can customize the Slack notification message in the `Config` node. * You can modify the reply email body in the `Gmail` node. We recommend including a scheduling link (e.g., to book a meeting). |
| Form ending            | form              | Display completion message to user    | Send a message          | —                         |                                                                                                                                 |
| ContactWebhook         | webhook           | Receive Slack button callback          | —                      | EmailContent              |                                                                                                                                 |
| EmailContent           | code              | Extract and validate email parameters  | ContactWebhook          | Send a email              |                                                                                                                                 |
| Send a email           | gmail             | Send reply email to inquiry sender     | EmailContent            | —                         |                                                                                                                                 |
| Sticky Note            | stickyNote        | Documentation                         | —                      | —                         |                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: `formTrigger`  
   - Configure the form:  
     - Title: "Inquiry Form"  
     - Description: "Thanks for visiting! You can contact me here."  
     - Fields:  
       - Name (required, placeholder "your name")  
       - Email address (required, placeholder "replace-me@example.com")  
       - Meesage (textarea, required, placeholder "request for me")  
   - This node triggers the workflow on new form entries.

2. **Create "Config" node**  
   - Type: `set`  
   - Assign variables by expressions:  
     - `name` = `{{$json["Name"]}}`  
     - `emailAddress` = `{{$json["Email address"]}}`  
     - `message` = `{{$json.Meesage}}`  
     - `slackMessage` =  
       ```
       :envelope: New Inquiry Received
       ```
       Name:  {{ $json['Name'] }}
       Email address: {{ $json['Email address'] }}
       Message: {{ $json.Meesage }}
       ```  
     - `contactWebhookUrl` = `"https://your.app.n8n.cloud/webhook/uuid"` (replace with actual production webhook URL)  
   - Connect output from "On form submission".

3. **Create "NewRecord" node**  
   - Type: `code`  
   - JavaScript code:  
     ```javascript
     const item = $input.first().json;
     return [{
       json: {
         name: item.name,
         emailAddress: item.emailAddress,
         message: item.message,
       }
     }];
     ```  
   - Connect output from "Config".

4. **Create "Append row in inquiry list" node**  
   - Type: `googleSheets`  
   - Operation: append  
   - Document ID: [Your Google Sheet ID]  
   - Sheet Name: "gid=0" (first sheet)  
   - Columns: Map automatically to input fields, including at least "Email address", "Meesage".  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Connect output from "NewRecord".

5. **Create "Send a message" node**  
   - Type: `slack`  
   - Channel ID: Set to your Slack channel ID where notifications should appear.  
   - Message Type: block  
   - Blocks (use JSON editor):  
     ```json
     {
       "blocks": [
         {
           "type": "section",
           "text": {
             "type": "mrkdwn",
             "text": "{{ $('Config').item.json.slackMessage.replaceAll('\\n', '\\\\n') }}"
           }
         },
         { "type": "divider" },
         {
           "type": "actions",
           "elements": [
             {
               "type": "button",
               "text": {
                 "type": "plain_text",
                 "text": "Contact",
                 "emoji": true
               },
               "value": "click_contact",
               "url": "{{ $('Config').item.json.contactWebhookUrl }}?name={{ $('Config').item.json.name }}&emailAddress={{ $('Config').item.json.emailAddress }}"
             }
           ]
         }
       ]
     }
     ```  
   - Credentials: Set Slack API OAuth credentials.  
   - Connect output from "Append row in inquiry list".

6. **Create "Form ending" node**  
   - Type: `form`  
   - Operation: completion  
   - Completion Title: "Thank you for contacting us."  
   - Completion Message: "We have received your message and will get back to you shortly."  
   - Connect output from "Send a message".

7. **Create "ContactWebhook" node**  
   - Type: `webhook`  
   - Path: Use a unique string or UUID (e.g., "0b9182aa-4014-4044-a9c2-3c993e69d643")  
   - Authentication: Basic Auth enabled  
   - Credentials: Configure Basic Auth credentials (username/password)  
   - This node listens to requests from the Slack "Contact" button.

8. **Create "EmailContent" node**  
   - Type: `code`  
   - JavaScript code:  
     ```javascript
     const raw = $input.first().json;
     const { name, emailAddress } = raw.query;

     if (!name || !emailAddress) {
       return [];
     }

     return [{
       json: {
         name,
         emailAddress
       }
     }];
     ```  
   - Connect output from "ContactWebhook".

9. **Create "Send a email" node**  
   - Type: `gmail`  
   - Send To: `={{ $json.emailAddress }}`  
   - Subject: "We received your message"  
   - Message:  
     ```
     Dear {{ $json.name }},

     Thank you for reaching out to us.
     We have received your inquiry and will get back to you shortly.

     Best regards,

     The Support Team
     ```  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - Connect output from "EmailContent".

10. **Connect all nodes** according to the flow:  
    - On form submission → Config → NewRecord → Append row in inquiry list → Send a message → Form ending  
    - ContactWebhook → EmailContent → Send a email

11. **Update all placeholder values:**  
    - Slack channel ID in "Send a message"  
    - Google Sheets document ID in "Append row in inquiry list"  
    - `contactWebhookUrl` in "Config" to the production URL of "ContactWebhook" node  
    - Credentials for Gmail, Slack, Google Sheets, and Basic Auth for webhook.

12. **Test entire workflow:**  
    - Submit the form.  
    - Verify Google Sheets row appended.  
    - Check Slack message appears with "Contact" button.  
    - Click "Contact" button to trigger webhook and send email reply.  
    - Confirm form completion message appears to user.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is triggered when the contact form is submitted. It automatically saves inquiry details to Google Sheets and notifies Slack. You can reply via Slack button to send an email reply.                                                                                                                                          | Sticky Note content within the workflow.                                                          |
| You can create a Slack App for API access and messaging here: https://api.slack.com/apps                                                                                                                                                                                                                                                    | Slack node credential setup.                                                                       |
| It is recommended to include a scheduling link in the email reply for better user engagement.                                                                                                                                                                                                                                              | Gmail node customization advice.                                                                  |
| Ensure Basic Auth credentials for the webhook are securely stored and managed.                                                                                                                                                                                                                                                              | ContactWebhook node security.                                                                      |
| Google Sheets document must have a sheet with appropriate columns matching the form data fields.                                                                                                                                                                                                                                            | Google Sheets node setup.                                                                          |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. All data processed is legal and public. Content strictly respects current content policies and contains no illegal, offensive, or protected elements.