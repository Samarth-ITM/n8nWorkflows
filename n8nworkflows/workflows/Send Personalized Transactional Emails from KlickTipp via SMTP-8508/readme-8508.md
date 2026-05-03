Send Personalized Transactional Emails from KlickTipp via SMTP

https://n8nworkflows.xyz/workflows/send-personalized-transactional-emails-from-klicktipp-via-smtp-8508


# Send Personalized Transactional Emails from KlickTipp via SMTP

### 1. Workflow Overview

This workflow automates the sending of personalized transactional emails triggered by KlickTipp outbound rules. It is designed for marketers or customer service teams using KlickTipp who want to send dynamic, branded emails through any SMTP-compatible service and track delivery status directly within KlickTipp.

**Target Use Cases:**  
- Sending confirmation, update, welcome, or announcement emails immediately after a KlickTipp event (e.g., tag applied).  
- Personalizing emails dynamically based on subscriber data fields.  
- Capturing and updating email delivery success or failure status back into KlickTipp for monitoring.

**Logical Blocks:**  
- **1.1 Input Reception:** Receive subscriber data from KlickTipp outbound webhook trigger.  
- **1.2 HTML Email Generation:** Build a personalized HTML email template with subscriber-specific data.  
- **1.3 SMTP Email Sending:** Deliver the generated email via SMTP using configured credentials.  
- **1.4 Delivery Status Update:** Update the KlickTipp subscriber record with “Sent” or “Failed” status depending on email sending outcome.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming webhook calls from KlickTipp outbound rules, receiving subscriber email and custom fields to personalize the message.

- **Nodes Involved:**  
  - Recieve the data from KlickTipp

- **Node Details:**  
  - **Node Name:** Recieve the data from KlickTipp  
  - **Type & Role:** KlickTipp Trigger node; entry point that activates workflow upon KlickTipp outbound event.  
  - **Configuration:** Uses KlickTipp API credentials; listens for outbound webhook calls.  
  - **Expressions/Variables:** Outputs JSON including subscriber email, first name, phone, website fields as received.  
  - **Input:** None (trigger).  
  - **Output:** Passes subscriber JSON data to next node.  
  - **Version:** Version 1 (community node).  
  - **Potential Failures:** Authentication errors with KlickTipp API; webhook misconfiguration; missing expected fields in payload.  
  - **Sub-workflow:** None.

#### 1.2 HTML Email Generation

- **Overview:**  
  Constructs a personalized HTML email using subscriber data fields, providing a branded, clean template.

- **Nodes Involved:**  
  - Generate HTML template

- **Node Details:**  
  - **Node Name:** Generate HTML template  
  - **Type & Role:** HTML node; generates HTML content for the email body.  
  - **Configuration:** Static HTML template with inline CSS; uses n8n expressions to insert dynamic subscriber fields such as first name, website, and phone.  
  - **Key Expressions:**  
    - `{{ $json.CustomFieldFirstName || 'there' }}` for salutation fallback.  
    - `{{ $json.fieldWebsite }}` and `{{ $json.CustomFieldWebsite }}` for website link.  
    - `{{ $json.CustomFieldPhone }}` for phone number.  
  - **Input:** Subscriber JSON data from KlickTipp trigger.  
  - **Output:** JSON including generated `html` field for email content.  
  - **Version:** 1.2  
  - **Potential Failures:** Missing or malformed subscriber fields causing broken HTML or empty placeholders; expression syntax errors.

#### 1.3 SMTP Email Sending

- **Overview:**  
  Sends the personalized HTML email to the subscriber via SMTP with defined sender and subject.

- **Nodes Involved:**  
  - Send email

- **Node Details:**  
  - **Node Name:** Send email  
  - **Type & Role:** Email Send node; transmits the email through configured SMTP credentials.  
  - **Configuration:**  
    - Uses SMTP credentials configured for Gmail SMTP server.  
    - Sends to subscriber email `={{ $('Recieve the data from KlickTipp').item.json.email }}`.  
    - From email fixed as `"KlickTipp team" no-reply@digital-results-international.com`.  
    - Subject set to “Thank you!”.  
    - HTML body from previous node’s `html` field.  
  - **Error Handling:** Set to continue on error, so workflow proceeds to update status accordingly.  
  - **Input:** JSON containing `html` from Generate HTML template node.  
  - **Output:** Passes success or error to next nodes.  
  - **Version:** 2.1  
  - **Potential Failures:** SMTP authentication failure; network timeouts; invalid recipient email; provider rate limits; malformed HTML causing rejection.

#### 1.4 Delivery Status Update

- **Overview:**  
  Updates the subscriber’s custom field in KlickTipp to reflect the email delivery status: “Sent” or “Failed,” along with the HTML content sent.

- **Nodes Involved:**  
  - Email delivery status: Sent  
  - Email delivery status: Failed

- **Node Details:**  

  - **Node Name:** Email delivery status: Sent  
    - **Type & Role:** KlickTipp node; updates subscriber custom fields on successful email delivery.  
    - **Configuration:**  
      - Sets “Email delivery status” field to “Sent”.  
      - Updates field with the actual HTML content sent.  
      - Uses subscriber email from trigger node for lookup.  
    - **Input:** Success output of Send email node.  
    - **Output:** None further.  
    - **Version:** 3  
    - **Potential Failures:** KlickTipp API errors; incorrect field IDs; subscriber email missing.

  - **Node Name:** Email delivery status: Failed  
    - **Type & Role:** KlickTipp node; updates subscriber custom fields on failed email delivery.  
    - **Configuration:**  
      - Sets “Email delivery status” field to “Failed”.  
      - Updates field with the attempted HTML content.  
      - Uses subscriber email from trigger node for lookup.  
    - **Input:** Error output of Send email node.  
    - **Output:** None further.  
    - **Version:** 3  
    - **Potential Failures:** Same as above.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                  | Input Node(s)                | Output Node(s)                          | Sticky Note                                                                                                          |
|-----------------------------|----------------------------------|--------------------------------|-----------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Recieve the data from KlickTipp | KlickTipp Trigger (community)     | Input Reception from KlickTipp | —                           | Generate HTML template                 | ## 1. Get data.                                                                                                       |
| Generate HTML template       | HTML Node                        | Create personalized HTML email | Recieve the data from KlickTipp | Send email                            | ## 2. Create an email body.                                                                                           |
| Send email                  | SMTP Email Send                  | Send email via SMTP             | Generate HTML template        | Email delivery status: Sent, Email delivery status: Failed | ## 3. Send an email.                                                                                                  |
| Email delivery status: Sent  | KlickTipp Update (community)    | Mark email as Sent in KlickTipp | Send email (success output)  | —                                     | ## 4. Update email delivery status.                                                                                   |
| Email delivery status: Failed| KlickTipp Update (community)    | Mark email as Failed in KlickTipp | Send email (error output)    | —                                     | ## 4. Update email delivery status.                                                                                   |
| Sticky Note                 | Sticky Note                     | Documentation / Notes           | —                           | —                                     | Community Node Disclaimer and detailed workflow explanation.                                                         |
| Sticky Note1                | Sticky Note                     | Documentation / Notes           | —                           | —                                     | ## 2. Create an email body.                                                                                            |
| Sticky Note2                | Sticky Note                     | Documentation / Notes           | —                           | —                                     | ## 3. Send an email.                                                                                                   |
| Sticky Note3                | Sticky Note                     | Documentation / Notes           | —                           | —                                     | ## 4. Update email delivery status.                                                                                   |
| Sticky Note4                | Sticky Note                     | Documentation / Notes           | —                           | —                                     | Community Node Disclaimer and detailed setup instructions, usage, and notes.                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create KlickTipp Trigger Node:**  
   - Add a KlickTipp Trigger node (community node).  
   - Authenticate with KlickTipp API credentials.  
   - Set to receive webhook calls from KlickTipp Outbound rule.  
   - This node will receive subscriber data including email and custom fields.

2. **Create HTML Node for Email Template:**  
   - Add an HTML node.  
   - Paste the provided HTML template into the editor.  
   - Use expressions to inject subscriber data, e.g., `{{ $json.CustomFieldFirstName || 'there' }}`, `{{ $json.CustomFieldWebsite }}`, `{{ $json.CustomFieldPhone }}`.  
   - Connect the output of KlickTipp Trigger node to this node.

3. **Create SMTP Send Email Node:**  
   - Add a Send Email node.  
   - Configure SMTP credentials (e.g., Gmail SMTP): provide host, port (465 or 587), username, password/app password, SSL/TLS settings.  
   - Set “To Email” to `={{ $('Recieve the data from KlickTipp').item.json.email }}`.  
   - Set “From Email” to `"KlickTipp team" no-reply@digital-results-international.com`.  
   - Set subject to “Thank you!”.  
   - Use the `html` output from the HTML node as the email body.  
   - Set error handling to “Continue on error”.  
   - Connect the output of the HTML node to this node.

4. **Create KlickTipp Update Node for Success:**  
   - Add a KlickTipp node.  
   - Authenticate with KlickTipp API credentials.  
   - Configure it to update subscriber resource.  
   - Set custom fields:  
     - “Email delivery status” field to “Sent”.  
     - Another field to store the HTML content (`={{ $('Generate HTML template').item.json.html }}`).  
   - Use subscriber email from trigger node for lookup.  
   - Connect the **success** output (first main) of Send Email node to this node.

5. **Create KlickTipp Update Node for Failure:**  
   - Duplicate the above KlickTipp update node.  
   - Change the “Email delivery status” field value to “Failed”.  
   - Use same HTML content for field update.  
   - Connect the **error** output (second main) of Send Email node to this node.

6. **Add Sticky Notes:**  
   - Add sticky notes with the provided content to document the workflow logic, disclaimers, and setup instructions.

7. **Activate the Workflow:**  
   - Ensure all credentials are properly configured and tested.  
   - Activate the workflow to listen for inbound KlickTipp outbound calls.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Community Node Disclaimer: This workflow uses KlickTipp community nodes. Automates transactional emails from KlickTipp via SMTP with personalization, dynamic HTML, delivery status updates, and flexible SMTP provider support.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Included as Sticky Note4 in workflow.                                                                              |
| How It Works and Setup Instructions: Step-by-step guide on installing KlickTipp nodes, configuring SMTP credentials, importing HTML template, activating workflow, and testing with KlickTipp tags.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Included as Sticky Note4 in workflow.                                                                              |
| Benefits: Immediate personalized communication, consistent branding, clear observability, flexible extensibility.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Included as Sticky Note4 in workflow.                                                                              |
| Testing and Deployment: Use KlickTipp tags to trigger outbound rules, verify email receipt and delivery status, review logs, adjust field mappings.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Included as Sticky Note4 in workflow.                                                                              |
| SMTP Provider Settings: Refer to your email provider’s documentation for correct SMTP host, port, and security settings (e.g., Gmail, Outlook).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | General recommendation for SMTP configuration.                                                                     |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and public.