Automated Form Response System with Google Sheets, Slack, Gmail & Contacts

https://n8nworkflows.xyz/workflows/automated-form-response-system-with-google-sheets--slack--gmail---contacts-6087


# Automated Form Response System with Google Sheets, Slack, Gmail & Contacts

### 1. Workflow Overview

This workflow automates the processing of new form responses recorded in a Google Sheets spreadsheet. Upon detection of a new row (representing a form submission), it performs three main actions:

- Notifies a Slack channel with details of the new entry.
- Sends a personalized acknowledgment email to the submitter via Gmail.
- Creates a new contact entry in Google Contacts using the submitted information.

**Target Use Cases:**  
- Customer support ticket creation and acknowledgment.  
- Automated team notifications for new client requests.  
- Maintaining an up-to-date contact list from form submissions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Detect new form submission in Google Sheets.  
- **1.2 Team Notification:** Send a formatted message to Slack channel.  
- **1.3 Customer Acknowledgment:** Send personalized email response via Gmail.  
- **1.4 Contact Management:** Create or update contact in Google Contacts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new rows added to a specific Google Sheets document and sheet, triggering downstream actions.

**Nodes Involved:**  
- Google Sheets Trigger

**Node Details:**  
- **Google Sheets Trigger**  
  - *Type:* Trigger node for Google Sheets.  
  - *Role:* Detects new rows added to the configured spreadsheet and sheet.  
  - *Configuration:*  
    - Document ID: `1xJn861StZtmTop8c2WBqf4SvGziT0use3zVCjT4n9-o`  
    - Sheet Name: `Sheet1` (GID=0)  
    - Polling interval: every minute.  
  - *Expressions:* None specifically; outputs the entire new row as JSON with keys such as Name, Email, Phone Number, Description.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connected to Slack notification node.  
  - *Edge Cases/Potential Failures:*  
    - Authentication errors with Google API.  
    - Polling delays or API quota limitations.  
    - New rows missing expected columns or malformed data.

---

#### 1.2 Team Notification

**Overview:**  
Sends a detailed notification message to a predefined Slack channel to alert team members of the new form entry.

**Nodes Involved:**  
- Send a message1 (Slack node)

**Node Details:**  
- **Send a message1**  
  - *Type:* Slack node for sending messages.  
  - *Role:* Posts notification messages to a Slack channel.  
  - *Configuration:*  
    - Channel ID: `C095F7HDKUL` (predefined channel)  
    - Message text includes fields: Name, Email, Phone Number, Problem Description dynamically inserted from Google Sheets data using expressions like `{{ $json.Name }}`.  
  - *Expressions:* Uses mustache templates referencing `$json` data from the trigger node.  
  - *Inputs:* Receives data from Google Sheets Trigger.  
  - *Outputs:* Connected to Gmail send message node.  
  - *Edge Cases/Potential Failures:*  
    - Slack API authentication or permission errors.  
    - Invalid or missing channel ID.  
    - Message formatting errors if data fields are empty or malformed.  
  - *Credential:* Slack API OAuth2 credentials configured.

---

#### 1.3 Customer Acknowledgment

**Overview:**  
Sends a personalized email to the form submitter acknowledging receipt of their problem report.

**Nodes Involved:**  
- Send a message (Gmail node)

**Node Details:**  
- **Send a message**  
  - *Type:* Gmail node for sending emails.  
  - *Role:* Sends a text email to the submitter’s email address.  
  - *Configuration:*  
    - Recipient: Email extracted dynamically from the Google Sheets Trigger node (`={{ $('Google Sheets Trigger').item.json.Email }}`).  
    - Subject: Fixed string "Ticket Created.!"  
    - Message body: Template text with dynamic placeholders for Name and Description, e.g., `Hey {{ Name }}` and the problem description.  
  - *Expressions:* Uses mustache syntax to interpolate fields from Google Sheets data.  
  - *Inputs:* Receives data from Slack node (Send a message1).  
  - *Outputs:* Connected to Google Contacts node.  
  - *Edge Cases/Potential Failures:*  
    - Gmail OAuth2 token expiration or permission issues.  
    - Invalid email addresses causing send failure.  
    - Message template errors if fields are missing.  
  - *Credential:* Gmail OAuth2 account configured.

---

#### 1.4 Contact Management

**Overview:**  
Creates a new contact in Google Contacts using the submitted name and phone number for future reference.

**Nodes Involved:**  
- Create a contact (Google Contacts node)

**Node Details:**  
- **Create a contact**  
  - *Type:* Google Contacts node.  
  - *Role:* Adds new contact entries with name and phone details.  
  - *Configuration:*  
    - Given Name and Family Name: Both set to the submitter’s Name from Google Sheets.  
    - Phone Number: Added as a mobile phone type using the submitted phone number.  
  - *Expressions:* Uses expressions to pull data from Google Sheets Trigger node, e.g., `={{ $('Google Sheets Trigger').item.json.Name }}` and phone number field.  
  - *Inputs:* Receives data from Gmail node (Send a message).  
  - *Outputs:* None (end of workflow).  
  - *Edge Cases/Potential Failures:*  
    - Google Contacts API quota or authentication issues.  
    - Duplicate contacts if not handled externally.  
    - Phone number format inconsistencies.  
  - *Credential:* Google Contacts OAuth2 account configured.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role          | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|---------------------|---------------------|-------------------------|-------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Google Sheets Trigger| Google Sheets Trigger| Input Reception Trigger | -                       | Send a message1         |                                                                                                |
| Send a message1      | Slack               | Team Notification       | Google Sheets Trigger   | Send a message          |                                                                                                |
| Send a message       | Gmail               | Customer Acknowledgment | Send a message1         | Create a contact        |                                                                                                |
| Create a contact     | Google Contacts     | Contact Management      | Send a message          | -                       |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node:**  
   - Type: Google Sheets Trigger  
   - Set credential to your Google Sheets OAuth2 account.  
   - Configure with:  
     - Document ID: `1xJn861StZtmTop8c2WBqf4SvGziT0use3zVCjT4n9-o`  
     - Sheet Name or GID: `Sheet1` or `gid=0`  
     - Polling interval: every minute.  
   - No inputs; output will be the new row’s JSON data.

2. **Create Slack Send Message node:**  
   - Type: Slack  
   - Set credential to Slack OAuth2 account.  
   - Configure:  
     - Channel: Select or enter channel ID `C095F7HDKUL`  
     - Message Text (use mustache expressions):  
       ```
       => Hello Team,A new entry has been added to the google sheets Node. The details are as follows.
       Name : {{ $json.Name }}
       Email : {{ $json.Email }}
       Phone Number : {{ $json['Phone Number']}}
       Problem Desc : {{ $json.Description }}

       Please handle the client's request as stated in the problem.
       ```
   - Connect Google Sheets Trigger node output to this node input.

3. **Create Gmail Send Message node:**  
   - Type: Gmail  
   - Set credential to Gmail OAuth2 account.  
   - Configure:  
     - Send To: `={{ $('Google Sheets Trigger').item.json.Email }}`  
     - Subject: `Ticket Created.!`  
     - Message (text only):  
       ```
       Hey  {{ $('Google Sheets Trigger').item.json.Name }}!

       We see that your Problem is "{{ $('Google Sheets Trigger').item.json.Description }}"
       We got your query and our team is working on it. We assure you that the problem will be solved Really Soon.

       Reach out to us in case of any help

       Thank you
       Regards,
       N8n Team
       ```
   - Connect Slack node output to this node input.

4. **Create Google Contacts node:**  
   - Type: Google Contacts  
   - Set credential to Google Contacts OAuth2 account.  
   - Configure:  
     - Given Name: `={{ $('Google Sheets Trigger').item.json.Name }}`  
     - Family Name: `={{ $('Google Sheets Trigger').item.json.Name }}` (same as given name)  
     - Phone Number: Add as mobile type: `={{ "" + $('Google Sheets Trigger').item.json["Phone Number"] }}`  
   - Connect Gmail node output to this node input.

5. **Activate the workflow:**  
   - Ensure all credentials are authorized and have required scopes.  
   - Test by adding a new row in the Google Sheet with appropriate columns: Name, Email, Phone Number, Description.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow is designed to automate customer support ticket acknowledgment and internal team notifications. | Workflow Purpose                                 |
| Ensure your Google Sheets columns exactly match the expected keys: Name, Email, Phone Number, Description.    | Data Consistency Requirement                      |
| Slack channel ID must be valid and the bot must have permissions to post messages in that channel.            | Slack API Permissions                             |
| Gmail OAuth2 credential requires permissions to send emails on behalf of the user.                            | Gmail API Setup                                  |
| Google Contacts OAuth2 credential requires contact creation permissions.                                       | Google Contacts API Setup                         |

---

**Disclaimer:**  
This document is generated exclusively from an automated n8n workflow export. It adheres strictly to content policies and does not contain any illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.