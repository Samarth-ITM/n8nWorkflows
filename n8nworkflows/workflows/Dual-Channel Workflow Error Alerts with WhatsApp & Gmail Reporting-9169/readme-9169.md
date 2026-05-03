Dual-Channel Workflow Error Alerts with WhatsApp & Gmail Reporting

https://n8nworkflows.xyz/workflows/dual-channel-workflow-error-alerts-with-whatsapp---gmail-reporting-9169


# Dual-Channel Workflow Error Alerts with WhatsApp & Gmail Reporting

---

### 1. Workflow Overview

This workflow is designed to automatically handle errors occurring in other n8n workflows by sending dual-channel alerts via WhatsApp and Gmail email. Its primary use case is to provide timely and documented notifications about workflow failures to designated recipients, ensuring rapid response and traceability.

The workflow is logically divided into two main blocks:

- **1.1 Triggering and Preparation Phase ("Get informations")**  
  Captures error events, extracts and prepares recipient contact information, and formats the data for separate sending.

- **1.2 Alert and Communication Phase ("Send alert")**  
  Sends notifications first through WhatsApp for immediacy, then after a delay sends an email as a formal record.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggering and Preparation Phase ("Get informations")

- **Overview:**  
  This block listens for workflow errors via an error trigger, then sets and formats the recipient data (email addresses and phone numbers) before splitting them into individual items for separate notification.

- **Nodes Involved:**  
  - Error Trigger  
  - set_recipient_EMAIL/NUMBER  
  - separate items

- **Node Details:**

  - **Error Trigger**  
    - *Type & Role:* Special n8n errorTrigger node designed to start this workflow automatically when an error occurs in any monitored workflow.  
    - *Configuration:* No parameters; triggers on any error from configured workflows.  
    - *Input/Output:* No input; outputs error metadata including workflow name, error message, last node executed, and recipient info placeholders.  
    - *Edge Cases:* If no workflows are linked to this as their error workflow, it will never trigger. If error metadata is incomplete, downstream expressions may fail.  
    - *Sub-workflow:* This node is the entry point activated by errors in other workflows.

  - **set_recipient_EMAIL/NUMBER**  
    - *Type & Role:* Set node that statically or dynamically assigns recipient emails and phone numbers, and re-maps workflow and error data for easy access downstream.  
    - *Configuration:* Defines variables:  
      - Email1, Email2: two recipient email addresses (default placeholder text "your_mail_adress")  
      - phone_number_1, phone_number_2: two recipient WhatsApp phone numbers (default placeholder text "your_phone_number")  
      - workflow.name, execution.error.message, execution.lastNodeExecuted: assigned from error trigger JSON for contextual info.  
    - *Input/Output:* Input from Error Trigger; outputs enriched JSON including contact and error info.  
    - *Edge Cases:* If emails or phone numbers are not updated from placeholders, notifications will fail or send to invalid recipients.

  - **separate items**  
    - *Type & Role:* Code node that splits a single input item containing two emails and two phone numbers into two separate items, each with one email and one phone number.  
    - *Configuration:* JavaScript code that:  
      - Validates input array is non-empty.  
      - Extracts Email1/phone_number_1 and Email2/phone_number_2 per input.  
      - Returns two separate items to enable parallel notification sending.  
    - *Key Variables:* Uses `$input.all()` to access all input data; references `Email1`, `Email2`, `phone_number_1`, `phone_number_2`.  
    - *Input/Output:* Input from set_recipient_EMAIL/NUMBER; outputs two items each with one email and one phone number.  
    - *Edge Cases:* Throws error if input is empty or not an array. If input JSON properties are missing, output items may be incomplete.

---

#### 1.2 Alert and Communication Phase ("Send alert")

- **Overview:**  
  This block sends the alert messages via WhatsApp immediately, waits to avoid hitting rate limits, then sends emails containing the error details.

- **Nodes Involved:**  
  - Send message (WhatsApp)  
  - Wait  
  - Send Email (Gmail)

- **Node Details:**

  - **Send message (WhatsApp)**  
    - *Type & Role:* WhatsApp node to send an immediate text alert to the phone number from the separated items.  
    - *Configuration:*  
      - Operation: send message  
      - Text body uses expressions to include workflow name and last node executed from current item JSON.  
      - Recipient phone number dynamically set from separated item `phone_number`.  
      - Uses WhatsApp API credentials ("WhatsApp Bfastai").  
    - *Input/Output:* Input from separate items node; outputs success/failure per message.  
    - *Edge Cases:* Authentication failures with WhatsApp API, invalid phone numbers, or message quota limits may cause errors.  
    - *Version:* TypeVersion 1.

  - **Wait**  
    - *Type & Role:* Wait node that introduces a delay after WhatsApp notification before sending email, to comply with API rate limits or pacing requirements.  
    - *Configuration:* No parameters set, meaning default behavior (typically no delay unless configured). Ideally should be configured with a delay duration.  
    - *Input/Output:* Input from Send message; output to Send Email.  
    - *Edge Cases:* Without configured delay, risk of flooding email service; if delay is too long, email notification is late.

  - **Send Email (Gmail)**  
    - *Type & Role:* Gmail node that sends the error alert email to the recipient email address from the separated items.  
    - *Configuration:*  
      - Recipient email dynamically assigned from separated item `Email`.  
      - Subject includes "⚠️workflow error" tag.  
      - Message body includes workflow name, last node executed, and detailed error message from item JSON.  
      - Email type: plain text.  
      - Uses Gmail OAuth2 credentials "Gmail account Client Web N8N AUTO".  
    - *Input/Output:* Input from Wait node; outputs email send status.  
    - *Edge Cases:* OAuth token expiration, quota limits, invalid email addresses, or network issues may cause email sending failures.  
    - *Version:* TypeVersion 2.1.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                           | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                              |
|---------------------------|--------------------|-----------------------------------------|-----------------------------|--------------------------|---------------------------------------------------------------------------------------------------------|
| Error Trigger             | errorTrigger       | Entry point; triggers on workflow errors | None                        | set_recipient_EMAIL/NUMBER | Covered with "Get informations" and "Triggering and Preparation Phase" descriptions                      |
| set_recipient_EMAIL/NUMBER | Set                | Sets recipient emails, phones, error info | Error Trigger               | separate items            | Covered with "Get informations" and "Triggering and Preparation Phase" descriptions                      |
| separate items            | Code               | Splits recipients into separate items    | set_recipient_EMAIL/NUMBER  | Send message              | Covered with "Get informations" and "Triggering and Preparation Phase" descriptions                      |
| Send message              | WhatsApp           | Sends alert message via WhatsApp          | separate items              | Wait                      | Covered with "Send alert" description                                                                   |
| Wait                      | Wait               | Delays to avoid API rate limits           | Send message                | Send Email                | Covered with "Send alert" description                                                                   |
| Send Email                | Gmail              | Sends formal error report email            | Wait                        | None                      | Covered with "Send alert" description                                                                   |
| Sticky Note               | StickyNote         | Displays "Get informations" header        | None                       | None                      |                                                                                                         |
| Sticky Note1              | StickyNote         | Displays "Send alert" header                | None                       | None                      |                                                                                                         |
| Sticky Note2              | StickyNote         | Describes phase 1: Triggering and Preparation | None                       | None                      | Explains Error Trigger, Set node, separate items node                                                   |
| Sticky Note3              | StickyNote         | Describes phase 2: Alert and Communication | None                       | None                      | Explains WhatsApp, Wait, and Gmail sending nodes                                                        |
| Sticky Note4              | StickyNote         | Instructions to link this workflow as error workflow in monitored workflows | None                       | None                      | Explains how to assign this error workflow to other workflows' Error Workflow field                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Error Trigger node**  
   - Type: errorTrigger  
   - No parameters needed  
   - This node will start the workflow automatically when any linked workflow fails.

2. **Create set_recipient_EMAIL/NUMBER node**  
   - Type: Set  
   - Configure to assign variables:  
     - Email1: your_mail_adress (replace with real email)  
     - Email2: your_mail_adress (replace with real email)  
     - phone_number_1: your_phone_number (replace with real number, international format)  
     - phone_number_2: your_phone_number (replace with real number)  
     - workflow.name: expression `={{ $json.workflow.name }}`  
     - execution.error.message: expression `={{ $json.execution.error.message }}`  
     - execution.lastNodeExecuted: expression `={{ $json.execution.lastNodeExecuted }}`  
   - Connect Error Trigger output to this node input.

3. **Create separate items node**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the provided JS code that:  
     - Validates input array  
     - Splits each input item into two items with one email and one phone number respectively  
   - Connect set_recipient_EMAIL/NUMBER output to this node input.

4. **Create Send message node (WhatsApp)**  
   - Type: WhatsApp  
   - Operation: send  
   - Text Body:  
     ```
     =The workflow "{{ $('separate items').item.json.workflow.name }}" encountered an error in the node "{{ $('separate items').item.json.execution.lastNodeExecuted }}"
     ```  
   - Recipient Phone Number: expression `={{ $('separate items').item.json.phone_number }}`  
   - Phone Number ID: your WhatsApp business phone number ID (e.g., "625419790655867")  
   - Credentials: Select or create WhatsApp API credentials (e.g., "WhatsApp Bfastai")  
   - Connect separate items output to this node input.

5. **Create Wait node**  
   - Type: Wait  
   - Configure a delay as needed (e.g., 5 seconds) to avoid API rate limits (optional but recommended)  
   - Connect Send message output to this node input.

6. **Create Send Email node (Gmail)**  
   - Type: Gmail  
   - Send To: expression `={{ $('separate items').item.json.Email }}`  
   - Subject: `=⚠️workflow error`  
   - Message:  
     ```
     =The workflow "{{ $('separate items').item.json.workflow.name }}" encountered an error in the node "{{ $('separate items').item.json.execution.lastNodeExecuted }}"

     Error message: {{ $('separate items').item.json.execution.error.message }}
     ```  
   - Email Type: text  
   - Credentials: Select or create Gmail OAuth2 credentials (e.g., "Gmail account Client Web N8N AUTO")  
   - Connect Wait output to this node input.

7. **Link the nodes in this order:**  
   - Error Trigger → set_recipient_EMAIL/NUMBER → separate items → Send message → Wait → Send Email

8. **(Optional) Add Sticky Note nodes** to label and document phases for clarity.

9. **Configure monitored workflows:**  
   - Open each workflow you want to monitor  
   - Go to Workflow Settings  
   - Set the "Error Workflow" field to this error workflow's name ("Error workflow")  
   - Save changes

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| To use this error workflow, assign it as the error workflow in other workflows' settings to automatically receive alerts.       | Sticky Note4 content; workflow settings in n8n                                                                       |
| WhatsApp API credentials must be properly configured for sending messages, including correct phone number ID and WhatsApp token. | Node "Send message" configuration                                                                                     |
| Gmail OAuth2 credentials require user authentication with Google and proper scopes for sending emails.                           | Node "Send Email" configuration                                                                                        |
| The separate items node uses JavaScript code that expects input data to contain specific keys; ensure set node outputs match.  | Node "separate items" explanation                                                                                      |
| Delay configuration in Wait node is recommended to avoid API rate limits but is not defined by default.                         | Node "Wait" description                                                                                                 |
| For monitoring multiple workflows, this error workflow can be reused by linking it as the error workflow field in each workflow. | Sticky Note4 content                                                                                                    |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---