Send Personalized HTML Welcome Emails to New Xero Contacts via SMTP

https://n8nworkflows.xyz/workflows/send-personalized-html-welcome-emails-to-new-xero-contacts-via-smtp-7587


# Send Personalized HTML Welcome Emails to New Xero Contacts via SMTP

### 1. Workflow Overview

This workflow automates the process of sending personalized HTML welcome emails to new contacts added in Xero, a cloud-based accounting software. It is designed to trigger when a new contact is created in Xero, verify that the contact is indeed new, fetch detailed information about the contact, generate a customized HTML email based on these details, and send the email via SMTP.

Logical blocks included in the workflow:

- **1.1 Input Reception:** Receives webhook notifications from Xero about new contacts.
- **1.2 New Contact Verification:** Checks if the webhook data corresponds to a genuinely new contact creation event.
- **1.3 Data Enrichment from Xero:** Retrieves full contact details using the Xero API.
- **1.4 Email Content Construction:** Builds a personalized HTML email using contact details.
- **1.5 Email Dispatch:** Sends the constructed email through the SMTP email node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives incoming webhook requests from Xero whenever a new contact event occurs.

- **Nodes Involved:**  
  - New Contact in Xero (Webhook)

- **Node Details:**  

  - **New Contact in Xero**  
    - Type: Webhook  
    - Role: Entry point for the workflow triggered by Xero webhook calls on contact events.  
    - Configuration: Listens for POST requests from Xero, capturing contact creation data payloads.  
    - Expressions/Variables: Receives raw webhook JSON payload from Xero.  
    - Input: External Xero webhook call.  
    - Output: Passes received data to the next node for evaluation.  
    - Version Notes: Version 2, standard webhook node.  
    - Edge Cases / Failures:  
      - Failure if the webhook is not properly configured or Xero does not send data.  
      - Potential for missing or malformed payload data.  
      - Network or authentication issues on webhook reception.  
    - Sub-workflow: None.

#### 2.2 New Contact Verification

- **Overview:**  
  Evaluates whether the received webhook data corresponds to a newly created contact (not an update or other event).

- **Nodes Involved:**  
  - Is it a NEW Contact? (IF node)

- **Node Details:**  

  - **Is it a NEW Contact?**  
    - Type: IF (Conditional Branch)  
    - Role: Filters workflow execution to continue only if the contact is newly created.  
    - Configuration: Condition checks fields in webhook data that indicate the creation event (e.g., status, event type).  
    - Expressions: Conditions likely reference `{{$json["event"]}}` or similar to detect new contact creation.  
    - Input: Data from the webhook node.  
    - Output: Two branches - "true" for new contacts, "false" to stop workflow or ignore.  
    - Version Notes: Version 2.2.  
    - Edge Cases / Failures:  
      - Incorrect condition logic could miss new contacts or process updates erroneously.  
      - Null or missing fields may cause expression evaluation errors.  
    - Sub-workflow: None.

#### 2.3 Data Enrichment from Xero

- **Overview:**  
  Fetches comprehensive contact details from Xero via API for use in personalized email content.

- **Nodes Involved:**  
  - Fetch Full Contact Details from Xero (Xero node)

- **Node Details:**  

  - **Fetch Full Contact Details from Xero**  
    - Type: Xero API node  
    - Role: Calls Xero API to retrieve detailed contact information using the contact ID from webhook.  
    - Configuration: Uses Xero API credentials; configured to perform a "Get Contact" operation with provided contact ID.  
    - Expressions: Uses `{{$json["contactId"]}}` or similar to specify which contact to fetch.  
    - Input: Contact ID from previous IF node output.  
    - Output: Detailed contact data forwarded to the email builder.  
    - Version Notes: Version 1.  
    - Edge Cases / Failures:  
      - API authentication errors (expired token, invalid credentials).  
      - Rate limiting by Xero API.  
      - Contact ID missing or invalid causing failed API calls.  
    - Sub-workflow: None.

#### 2.4 Email Content Construction

- **Overview:**  
  Builds a personalized HTML welcome email using the detailed contact data.

- **Nodes Involved:**  
  - Build Personalized HTML Email (Code node)

- **Node Details:**  

  - **Build Personalized HTML Email**  
    - Type: Code (JavaScript)  
    - Role: Generates HTML email content, dynamically inserting contact-specific data such as name, company, and other details.  
    - Configuration: Custom JavaScript code that accesses incoming JSON data and constructs HTML string with placeholders replaced.  
    - Expressions: Uses `items[0].json` or direct JSON access for contact properties.  
    - Input: Detailed contact info JSON from Xero node.  
    - Output: Object with HTML content and email metadata (subject, recipient address).  
    - Version Notes: Version 2.  
    - Edge Cases / Failures:  
      - Code errors if expected fields are missing or malformed.  
      - Encoding or HTML injection risks if input data is not sanitized.  
    - Sub-workflow: None.

#### 2.5 Email Dispatch

- **Overview:**  
  Sends the constructed HTML welcome email to the new contact using SMTP.

- **Nodes Involved:**  
  - Send Personalized Welcome Email (Email Send node)

- **Node Details:**  

  - **Send Personalized Welcome Email**  
    - Type: Email Send (SMTP)  
    - Role: Dispatches the personalized HTML email via configured SMTP credentials.  
    - Configuration: SMTP server details (host, port, credentials), email parameters such as "From", "To" (populated dynamically), "Subject", and body (HTML) are set.  
    - Expressions: Uses data from previous node for recipient email and HTML content.  
    - Input: Email content and recipient data from Code node.  
    - Output: Email delivery status.  
    - Version Notes: Version 2.1.  
    - Edge Cases / Failures:  
      - SMTP authentication errors, server unreachable.  
      - Email formatting issues causing delivery failure.  
      - Invalid recipient email addresses.  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                              | Input Node(s)             | Output Node(s)                  | Sticky Note                          |
|-------------------------------|--------------------|----------------------------------------------|---------------------------|---------------------------------|------------------------------------|
| New Contact in Xero            | Webhook            | Entry point, receives contact creation events | —                         | Is it a NEW Contact?             |                                    |
| Is it a NEW Contact?           | IF                 | Filters for new contact creation events       | New Contact in Xero        | Fetch Full Contact Details from Xero |                                    |
| Fetch Full Contact Details from Xero | Xero API          | Retrieves detailed contact data from Xero     | Is it a NEW Contact?       | Build Personalized HTML Email    |                                    |
| Build Personalized HTML Email  | Code               | Builds personalized HTML email content        | Fetch Full Contact Details from Xero | Send Personalized Welcome Email |                                    |
| Send Personalized Welcome Email | Email Send (SMTP)  | Sends HTML welcome email via SMTP              | Build Personalized HTML Email | —                               |                                    |
| Sticky Note                   | Sticky Note        | —                                            | —                         | —                               |                                    |
| Sticky Note1                  | Sticky Note        | —                                            | —                         | —                               |                                    |
| Sticky Note2                  | Sticky Note        | —                                            | —                         | —                               |                                    |
| Sticky Note3                  | Sticky Note        | —                                            | —                         | —                               |                                    |
| Sticky Note4                  | Sticky Note        | —                                            | —                         | —                               |                                    |
| Sticky Note5                  | Sticky Note        | —                                            | —                         | —                               |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: "New Contact in Xero"  
   - Type: Webhook  
   - Configure to listen for POST requests from Xero webhook on Contact creation events.  
   - No additional parameters are required.  
   - Save and activate webhook to obtain URL for Xero configuration.

2. **Create IF Node**  
   - Name: "Is it a NEW Contact?"  
   - Type: IF  
   - Connect input from "New Contact in Xero" node.  
   - Configure condition to check if the webhook payload indicates a new contact creation event.  
     - Example: Check if `{{$json["event"]}}` equals `"ContactCreated"` or similar field specific to Xero webhook.  
   - Set true output to proceed, false output to stop workflow.

3. **Create Xero Node**  
   - Name: "Fetch Full Contact Details from Xero"  
   - Type: Xero API  
   - Connect input from true output of "Is it a NEW Contact?" node.  
   - Configure credentials for Xero API (OAuth2 or API key as required).  
   - Set operation to "Get Contact".  
   - Use expression to supply Contact ID from webhook data, e.g., `{{$json["contactId"]}}`.  
   - Save configuration.

4. **Create Code Node**  
   - Name: "Build Personalized HTML Email"  
   - Type: Code (JavaScript)  
   - Connect input from "Fetch Full Contact Details from Xero".  
   - Write JavaScript code to:  
     - Extract necessary contact details (e.g., name, email, company).  
     - Construct an HTML string incorporating these details for a welcome message.  
     - Set output JSON with keys for email subject, recipient email, and HTML body.  
   - Example output object:  
     ```js
     return [{
       json: {
         to: contactEmail,
         subject: `Welcome to Our Service, ${contactName}!`,
         html: generatedHtmlContent,
       }
     }];
     ```

5. **Create Email Send Node**  
   - Name: "Send Personalized Welcome Email"  
   - Type: Email Send (SMTP)  
   - Connect input from "Build Personalized HTML Email".  
   - Configure SMTP credentials (host, port, username, password).  
   - Map "To" field to `{{$json["to"]}}`.  
   - Set "Subject" to `{{$json["subject"]}}`.  
   - Set email body type to HTML and set body to `{{$json["html"]}}`.  
   - Save configuration.

6. **Connect Workflow**  
   - Chain nodes as follows:  
     `New Contact in Xero` → `Is it a NEW Contact?` → (true) → `Fetch Full Contact Details from Xero` → `Build Personalized HTML Email` → `Send Personalized Welcome Email`

7. **Test and Activate**  
   - Activate webhook.  
   - Test by creating a new contact in Xero and verify email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                     |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------|
| This workflow requires proper setup of Xero webhook for contact creation events to trigger.  | Xero Developer Docs: https://developer.xero.com/  |
| SMTP credentials must allow external sending from n8n environment; consider using OAuth2 SMTP.| General SMTP setup instructions in n8n docs       |
| Personalization code should sanitize inputs to prevent HTML injection in emails.              | Security best practices for email templates         |
| Test with sandbox Xero environment if available to avoid sending emails during development.  | Xero sandbox environment info                       |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.