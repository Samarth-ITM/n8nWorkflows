Salesforce Lead Capture with GPT-4 Personalized Email & SMS Follow-Up

https://n8nworkflows.xyz/workflows/salesforce-lead-capture-with-gpt-4-personalized-email---sms-follow-up-6102


# Salesforce Lead Capture with GPT-4 Personalized Email & SMS Follow-Up

### 1. Workflow Overview

This workflow automates the capture and follow-up of web leads using n8n, Salesforce, OpenAI’s GPT-4, and communication channels (email and SMS). It targets businesses seeking a flexible alternative to Salesforce’s rigid Web-to-Lead forms by integrating AI-powered personalized messaging and multi-channel outreach based on user preferences.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Collects lead data via an n8n web form.
- **1.2 Salesforce Lead Creation:** Creates a new Lead record in Salesforce using the submitted form data.
- **1.3 AI-Powered Personalized Message Generation:** Uses OpenAI GPT-4 to craft a personalized welcome message tailored to the lead’s interests and contact preference.
- **1.4 Conditional Follow-up Dispatch:** Routes the personalized message to the appropriate communication channel (SMS or email) based on the lead’s stated preference.
- **1.5 Informational Notes:** Provides contextual documentation on the workflow’s purpose and advantages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Collects lead data through an embedded web form, capturing essential contact details and preferences.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing user input via a web form titled "Contact Us".  
  - Configuration:  
    - Fields: First Name (required), Last Name (required), Email (optional), Phone (SMS) (optional), Description (textarea), Contact Preference (dropdown with "Email" or "Phone", required).  
    - Submit button labeled "Submit".  
    - Form description: "We'll get back to you soon".  
  - Expressions: None directly, but form data fields are referenced downstream.  
  - Inputs: External web form submissions via webhook.  
  - Outputs: JSON object containing all user input fields.  
  - Edge Cases:  
    - Missing required fields causes form validation failure on client side.  
    - Optional fields may be empty; downstream nodes must handle potential nulls.  
    - Webhook availability critical; downtime causes lead loss.  
  - Version: 2.2

#### 2.2 Salesforce Lead Creation

- **Overview:**  
Creates a Salesforce Lead record using form data, mapping form fields to Salesforce Lead fields.

- **Nodes Involved:**  
  - Create Salesforce Lead

- **Node Details:**  

  **Create Salesforce Lead**  
  - Type: Salesforce node (OAuth2 authentication)  
  - Role: Creates a new Lead object in Salesforce with mapped form data.  
  - Configuration:  
    - Company set to "Web Lead {Last Name}" (dynamic expression).  
    - Last Name from form's "Last Name".  
    - Additional fields: Email, First Name, Description, Mobile Phone mapped respectively from form inputs.  
  - Expressions: Uses expressions to dynamically map inputs from form JSON.  
  - Inputs: From "On form submission" node.  
  - Outputs: Salesforce operation result; passed to OpenAI node.  
  - Credentials: Requires Salesforce OAuth2 credentials with Lead create permissions.  
  - Edge Cases:  
    - Salesforce auth failures or token expiration.  
    - Data validation errors if field mappings are incorrect or mandatory Salesforce fields missing.  
    - Salesforce API rate limits.  
  - Version: 1

#### 2.3 AI-Powered Personalized Message Generation

- **Overview:**  
Generates a personalized follow-up message tailored to lead information and contact preference using GPT-4.

- **Nodes Involved:**  
  - OpenAI

- **Node Details:**  

  **OpenAI**  
  - Type: OpenAI GPT-4 via Langchain node  
  - Role: Creates a customized welcome message based on lead data.  
  - Configuration:  
    - Model: GPT-4.1 (latest GPT-4 variant).  
    - Messages:  
      - User prompt: Requests a personalized welcome message using the entire form submission JSON stringified.  
      - System prompt: Defines assistant persona as a friendly, professional sales assistant specializing in automation, Salesforce, and n8n workflows.  
        - Provides instructions to adapt message format depending on contact preference:  
          - Email: Detailed, positive, action-oriented message under 200 words, includes greeting, body, call-to-action, and company signature ("Best regards, [Your Company Name] Team").  
          - Phone (SMS): Concise (<160 characters), direct, friendly, no signature or links, focused on engagement.  
        - Personalization includes first name, last name, and user description to tailor messaging and suggest specific help/packages.  
  - Expressions: Uses `"{{ JSON.stringify($('On form submission').item.json)}}"` to pass entire form data.  
  - Inputs: From "Create Salesforce Lead" node.  
  - Outputs: JSON containing the generated message content.  
  - Credentials: Requires OpenAI API key with GPT-4 access.  
  - Edge Cases:  
    - OpenAI API rate limits or downtime.  
    - Message generation failures or unexpected output format.  
    - Timeout if response delayed.  
  - Version: 1.8

#### 2.4 Conditional Follow-up Dispatch

- **Overview:**  
Routes the personalized message to the preferred contact method: SMS or Email.

- **Nodes Involved:**  
  - Switch  
  - Send SMS  
  - Send Email

- **Node Details:**  

  **Switch**  
  - Type: Switch node  
  - Role: Checks the "Contact Preference:" field from the form submission to route the message.  
  - Configuration:  
    - Two rules:  
      - If preference equals "Phone (SMS)", route to SMS sending node.  
      - If preference equals "Email", route to email sending node.  
  - Inputs: From OpenAI node.  
  - Outputs: Two outputs, one for SMS path, one for email path.  
  - Edge Cases:  
    - Unexpected or missing contact preference leads to no output or dead-end.  
    - Case sensitivity enforced strictly.  
  - Version: 3.2

  **Send SMS**  
  - Type: Twilio node  
  - Role: Sends SMS message using Twilio API.  
  - Configuration:  
    - To: Phone number from form submission.  
    - From: Configured Twilio phone number (placeholder "phoneNumber" in config).  
    - Message: Content from OpenAI generated message.  
  - Inputs: From Switch node (Phone SMS branch).  
  - Credentials: Twilio API credentials with SMS send permissions.  
  - Edge Cases:  
    - Invalid or missing phone number.  
    - Twilio API errors (auth, rate limits, invalid sender).  
    - Message length over SMS limit handled by OpenAI prompt (under 160 chars).  
  - Notes: Sticky note: "Send SMS using Twilio".  
  - Version: 1

  **Send Email**  
  - Type: Email Send node (SMTP)  
  - Role: Sends personalized email to lead.  
  - Configuration:  
    - To: Email from form submission.  
    - From: Placeholder "emailPlaceholder" (should be replaced with valid email address).  
    - Subject: "Welcome {First Name}".  
    - Text: OpenAI message content.  
    - Email format: Plain text.  
  - Inputs: From Switch node (Email branch).  
  - Credentials: SMTP credentials (e.g., company SMTP server).  
  - Edge Cases:  
    - Missing or invalid email address.  
    - SMTP connection failures or auth issues.  
    - Email deliverability concerns (spam filters).  
  - Version: 2.1

#### 2.5 Informational Notes

- **Overview:**  
Provides a sticky note explaining the workflow’s purpose and usage recommendations.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  

  **Sticky Note**  
  - Type: Documentation node  
  - Role: Describes workflow as an n8n alternative to Salesforce Web-to-Lead, emphasizing AI personalization and extensibility.  
  - Content Highlights:  
    - Captures leads via n8n form trigger.  
    - Creates Salesforce Lead records.  
    - Personalized responses generated by OpenAI.  
    - Conditional delivery by email or SMS.  
    - Suggests adding file upload fields for advanced AI processing (e.g., lead scoring, document analysis).  
  - Inputs/Outputs: None (informational only).  
  - Version: 1

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                         | Input Node(s)           | Output Node(s)           | Sticky Note                          |
|------------------------|----------------------------------|---------------------------------------|------------------------|--------------------------|------------------------------------|
| On form submission     | Form Trigger                     | Captures lead data via web form       | (Webhook external)      | Create Salesforce Lead    |                                    |
| Create Salesforce Lead | Salesforce                      | Creates Lead record in Salesforce     | On form submission      | OpenAI                   |                                    |
| OpenAI                 | OpenAI GPT-4 via Langchain       | Generates personalized message        | Create Salesforce Lead  | Switch                   |                                    |
| Switch                 | Switch                          | Routes message by contact preference  | OpenAI                  | Send SMS, Send Email      |                                    |
| Send SMS               | Twilio                         | Sends SMS to lead's phone             | Switch                  | (Terminal)               | Send SMS using Twilio              |
| Send Email             | Email Send (SMTP)               | Sends personalized email to lead      | Switch                  | (Terminal)               |                                    |
| Sticky Note            | Sticky Note                    | Documentation and workflow explanation| None                    | None                     | ## n8n Web Lead Form Alternative to Salesforce Web-to-Lead ... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node: "On form submission"**  
   - Node Type: Form Trigger  
   - Configure webhook with auto-generated webhook ID.  
   - Set form title: "Contact Us".  
   - Add form fields:  
     - First Name (required)  
     - Last Name (required)  
     - Email (email type, optional)  
     - Phone (SMS) (optional)  
     - Tell Us About Yourself (Description) (textarea, optional)  
     - Contact Preference (dropdown, required) with options: "Email" and "Phone".  
   - Set submit button label: "Submit".  
   - Set form description: "We'll get back to you soon".  
   - Save node.

2. **Add the Salesforce node: "Create Salesforce Lead"**  
   - Node Type: Salesforce  
   - Set operation to Create Lead.  
   - Map fields:  
     - Company: `Web Lead {{ $json["Last Name"] }}` (use expression)  
     - Last Name: `{{ $json["Last Name"] }}`  
     - Additional Fields:  
       - Email: `{{ $json.Email }}`  
       - First Name: `{{ $json["First Name"] }}`  
       - Description: `{{ $json["Tell Us About Yourself (Description)"] }}`  
       - Mobile Phone: `{{ $json["Phone (SMS)"] }}`  
   - Configure Salesforce OAuth2 credentials with permissions to create Leads.  
   - Connect "On form submission" node output to this node input.

3. **Add the OpenAI node: "OpenAI"**  
   - Node Type: OpenAI GPT-4 (via Langchain node)  
   - Model: Select "gpt-4.1" or latest GPT-4 model available.  
   - Messages:  
     - User message:  
       ```
       Create a personalized welcome message for this lead:
       {{ JSON.stringify($('On form submission').item.json)}}

       Tailor the message to their interest in booking an n8n form creation. Make it warm and helpful.
       ```  
     - System message:  
       ```
       You are a friendly, professional sales assistant for an automation and software development service specializing in n8n workflows, Salesforce integrations, and custom forms. Your goal is to create engaging, personalized welcome messages that respond directly to the user's description, encouraging them to proceed with booking or next steps.

       Adapt the message format based on the contact preference:
       - If preference is "Email": Make it detailed, positive, and action-oriented (under 200 words). Include a greeting, body with personalization, call-to-action (e.g., reply or schedule), and end with your company signature: "Best regards, [Your Company Name] Team" and without the Subject.
       - If preference is "Phone (SMS)": Keep it very concise (under 160 characters), friendly, and direct. Include a short greeting, key offer, and simple call-to-action. No signature or links; focus on quick engagement; without the Subject.

       Always personalize using the user's first name, last name, and description. If the description mentions booking or creating something (e.g., an n8n form), offer specific help and suggest packages.
       ```  
   - Set OpenAI API credentials with GPT-4 access.  
   - Connect "Create Salesforce Lead" node output to this node input.

4. **Add the Switch node: "Switch"**  
   - Node Type: Switch  
   - Configure two rules on `$('On form submission').item.json["Contact Preference:"]`:  
     - Rule 1: Equals "Phone (SMS)" (case sensitive) → output 1  
     - Rule 2: Equals "Email" (case sensitive) → output 2  
   - Connect "OpenAI" node output to this node input.

5. **Add the Twilio node: "Send SMS"**  
   - Node Type: Twilio  
   - Configure:  
     - To: `{{ $('On form submission').item.json["Phone (SMS)"] }}` (expression)  
     - From: Your Twilio phone number (replace "phoneNumber" placeholder)  
     - Message: `{{ $('OpenAI').item.json.message.content }}`  
   - Set Twilio API credentials with SMS sending enabled.  
   - Connect Switch output 1 (Phone SMS) to this node input.

6. **Add the Email Send node: "Send Email"**  
   - Node Type: Email Send (SMTP)  
   - Configure:  
     - To Email: `{{ $('On form submission').item.json.Email }}`  
     - From Email: Set a valid sender email address (replace "emailPlaceholder").  
     - Subject: `Welcome {{ $('On form submission').item.json["First Name"] }}`  
     - Text: `{{ $('OpenAI').item.json.message.content }}`  
     - Email format: Text (plain)  
   - Set SMTP credentials with sending permission.  
   - Connect Switch output 2 (Email) to this node input.

7. **Add a Sticky Note node for documentation**  
   - Node Type: Sticky Note  
   - Content:  
     ```
     ## n8n Web Lead Form Alternative to Salesforce Web-to-Lead

     This workflow captures leads via n8n's form trigger, creates Salesforce records, personalizes responses with OpenAI, and sends via email/SMS based on preference. Avoids rigid Salesforce Web-to-Lead by adding logic, AI, and extensibility. Add file upload fields for AI processing (e.g., document analysis) to unlock potentials like lead scoring or content insights.
     ```  
   - Place visually near start of workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| This workflow provides a flexible, AI-enhanced alternative to Salesforce Web-to-Lead forms, enabling personalized communication and multi-channel outreach. | Workflow purpose documentation       |
| Consider enhancing form with file upload fields to enable advanced AI processing such as lead scoring or document content insights.             | Sticky Note in workflow               |
| Twilio SMS sending requires a valid Twilio phone number with SMS capabilities and configured credential in n8n.                                  | Twilio setup reference                |
| SMTP "From Email" must be replaced with a verified sender address to ensure email deliverability.                                                | Email setup best practice             |
| OpenAI GPT-4 model access requires subscription and API key with GPT-4 enabled.                                                                  | OpenAI API documentation              |
| Salesforce OAuth2 credentials must have sufficient permissions to create Lead records and use Salesforce API.                                    | Salesforce integration guidelines     |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automation workflow. It complies with current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.