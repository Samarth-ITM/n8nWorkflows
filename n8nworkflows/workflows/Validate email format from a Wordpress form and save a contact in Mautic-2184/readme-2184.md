Validate email format from a Wordpress form and save a contact in Mautic

https://n8nworkflows.xyz/workflows/validate-email-format-from-a-wordpress-form-and-save-a-contact-in-mautic-2184


# Validate email format from a Wordpress form and save a contact in Mautic

### 1. Workflow Overview

This workflow automates the processing of leads collected via a Wordpress form by validating and formatting the incoming data before saving it as a contact in Mautic, a marketing automation tool. It includes data normalization, email validation, contact creation, and handling of invalid emails by adding those leads to a "Do Not Contact" list.

Logical blocks:

- **1.1 Input Reception:** Receiving lead data from a Wordpress form via a webhook.
- **1.2 Data Normalization:** Formatting lead data (name, email, mobile) and performing basic email validation.
- **1.3 Contact Creation:** Creating the lead as a contact in Mautic if the email is valid.
- **1.4 Invalid Email Handling:** Adding leads with invalid emails to Mautic’s Do Not Contact (DNC) list.
- **1.5 Workflow Completion:** Ending the workflow after processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives POST requests from a Wordpress form submission via an HTTP webhook. It acts as the entry point to the workflow, capturing the raw lead data.

**Nodes Involved:**  
- WordpressForm  
- Sticky Note (contextual note for this block)

**Node Details:**

- **WordpressForm**  
  - *Type:* Webhook  
  - *Role:* Listens for POST requests from Wordpress form submissions.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Path: Unique webhook ID (e.g., `917366ee-14a8-4fef-9f0b-6638cdc35fad`)  
  - *Key Expressions:* None (receives raw JSON payload)  
  - *Inputs:* External HTTP request  
  - *Outputs:* JSON containing form data under `$json.body`  
  - *Potential Failures:* No data received if webhook URL is not called; malformed data if form fields change unexpectedly.  
  - *Sticky Note:* "Receive Data from Wordpress Form - You can customize your form fields in the way that best suits your marketing campaigns."

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Provides user guidance for this block.

---

#### 1.2 Data Normalization

**Overview:**  
This block extracts and normalizes the lead data fields to prepare for validation and creation. It formats the name in title case, converts the email to lowercase, and validates the email format. It also trims extraneous data to optimize workflow performance.

**Nodes Involved:**  
- LeadData  
- Sticky Note1

**Node Details:**

- **LeadData**  
  - *Type:* Set node  
  - *Role:* Extracts and formats fields from the incoming payload.  
  - *Configuration:*  
    - Sets `name` to the title case version of `body.Nome` field.  
    - Sets `email` to lowercase `body['E-mail']`.  
    - Sets `mobile` from `body.WhatsApp`.  
    - Sets `form` from `body.form_id`.  
    - Sets `email_valid` as a boolean result of validating the email format with `$json.body['E-mail'].isEmail()`.  
  - *Key Expressions:*  
    - `={{ $json.body.Nome.toTitleCase() }}`  
    - `={{ $json.body['E-mail'].toLowerCase() }}`  
    - `={{ $json.body['E-mail'].isEmail() }}`  
  - *Inputs:* Output from WordpressForm  
  - *Outputs:* JSON with normalized fields and email validation flag  
  - *Potential Failures:* If expected fields are missing or have unexpected formats, expressions might fail or produce incorrect data.  
  - *Sticky Note:* "Normalize Data - Let's separate the data we are going to use and remove everything that is unnecessary..."

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Explains the purpose and benefits of data normalization and validation.

---

#### 1.3 Contact Creation

**Overview:**  
This block creates a contact in Mautic using the normalized data, but only if the email is valid. It maps the normalized fields to Mautic’s contact fields.

**Nodes Involved:**  
- CreateContactMautic  
- Sticky Note3

**Node Details:**

- **CreateContactMautic**  
  - *Type:* Mautic node  
  - *Role:* Creates a new contact in Mautic with the lead’s data.  
  - *Configuration:*  
    - Email: `{{$json.email}}`  
    - First Name: `{{$json.name}}`  
    - Additional field `mobile`: `{{$json.mobile}}`  
    - Credentials: Mautic API credentials (OAuth2 or API key configured under "Mautic account")  
  - *Inputs:* Output from LeadData node  
  - *Outputs:* Mautic contact creation response (including contact ID)  
  - *Potential Failures:* Authentication errors with Mautic credentials, API rate limits, invalid data rejection by Mautic  
  - *Sticky Note:* "Create Contact on Mautic - Create a contact in Mautic where you will map the fields you need."

- **Sticky Note3**  
  - *Type:* Sticky Note  
  - *Role:* Explains the contact creation purpose.

---

#### 1.4 Invalid Email Handling

**Overview:**  
If the email is invalid, this block adds the lead to Mautic’s Do Not Contact (DNC) list to avoid future communications.

**Nodes Involved:**  
- CheckEmailValid  
- LeadMauticDNC  
- Sticky Note2

**Node Details:**

- **CheckEmailValid**  
  - *Type:* If node  
  - *Role:* Checks whether the email is valid using the boolean flag from LeadData.  
  - *Configuration:*  
    - Condition: The field `email_valid` from LeadData must be `true`.  
  - *Inputs:* Output from CreateContactMautic  
  - *Outputs:*  
    - True branch: (no further action, workflow ends)  
    - False branch: Leads to LeadMauticDNC (add to DNC list)  
  - *Potential Failures:* Expression failure if `email_valid` is missing, logical errors if condition is misconfigured.

- **LeadMauticDNC**  
  - *Type:* Mautic node  
  - *Role:* Adds the contact to the Do Not Contact list with a reason and comment.  
  - *Configuration:*  
    - Operation: Edit Do Not Contact List  
    - Contact ID: `{{$json.id}}` (from Mautic contact creation response)  
    - Reason: Code “3” (predefined in Mautic for invalid email)  
    - Comments: “Did not pass basic email validation”  
    - Credentials: Same Mautic API credentials  
  - *Inputs:* False branch output from CheckEmailValid  
  - *Outputs:* Confirmation response from Mautic  
  - *Potential Failures:* Invalid contact ID if contact creation failed, authentication errors, API errors.

- **Sticky Note2**  
  - *Type:* Sticky Note  
  - *Role:* Describes the validation check and contact creation conditional logic.

---

#### 1.5 Workflow Completion

**Overview:**  
Final node simply marks the end of the workflow after processing all cases.

**Nodes Involved:**  
- End

**Node Details:**

- **End**  
  - *Type:* NoOp node (no operation)  
  - *Role:* Serves as a logical endpoint for workflow termination after invalid email handling.  
  - *Inputs:* Output from LeadMauticDNC  
  - *Outputs:* None  
  - *Potential Failures:* None

---

### 3. Summary Table

| Node Name          | Node Type        | Functional Role                         | Input Node(s)       | Output Node(s)          | Sticky Note                                                                                                  |
|--------------------|------------------|---------------------------------------|---------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| WordpressForm      | Webhook          | Receives data from Wordpress form     | (external HTTP)     | LeadData                | Receive Data from Wordpress Form: You can customize your form fields in the way that best suits your marketing campaigns. |
| Sticky Note        | Sticky Note      | Guidance on input reception            |                     |                         | Receive Data from Wordpress Form: You can customize your form fields in the way that best suits your marketing campaigns. |
| LeadData           | Set              | Normalize and validate incoming data  | WordpressForm       | CreateContactMautic      | Normalize Data: Separate needed data and apply formatting and validation using expressions.                   |
| Sticky Note1       | Sticky Note      | Guidance on data normalization         |                     |                         | Normalize Data: Separate needed data and apply formatting and validation using expressions.                   |
| CreateContactMautic| Mautic           | Create contact in Mautic               | LeadData            | CheckEmailValid          | Create Contact on Mautic: Map fields needed for contact creation.                                             |
| Sticky Note3       | Sticky Note      | Guidance on contact creation            |                     |                         | Create Contact on Mautic: Map fields needed for contact creation.                                             |
| CheckEmailValid    | If               | Branch based on email validity         | CreateContactMautic | LeadMauticDNC (False)   | Checks if email can be valid to create contact in Mautic with correct registration information.               |
|                    |                  |                                       |                     | (True branch ends)       | Checks if email can be valid to create contact in Mautic with correct registration information.               |
| LeadMauticDNC      | Mautic           | Add invalid email leads to DNC list    | CheckEmailValid (F) | End                     | Checks if email can be valid to create contact in Mautic with correct registration information.               |
| End                | NoOp             | Terminates workflow                     | LeadMauticDNC       |                         |                                                                                                              |
| Sticky Note2       | Sticky Note      | Guidance on email validation handling  |                     |                         | Checks if email can be valid to create contact in Mautic with correct registration information.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**  
   - Add a **Webhook** node named `WordpressForm`.  
   - Configure HTTP Method: `POST`.  
   - Set the path to a unique identifier (e.g., `917366ee-14a8-4fef-9f0b-6638cdc35fad`).  
   - This node will receive form data submissions from Wordpress.

2. **Add a Set Node for Data Normalization**  
   - Add a **Set** node named `LeadData`.  
   - Connect input from `WordpressForm`.  
   - Clear existing fields.  
   - Add fields with these expressions:  
     - `name`: `={{ $json.body.Nome.toTitleCase() }}`  
     - `email`: `={{ $json.body['E-mail'].toLowerCase() }}`  
     - `mobile`: `={{ $json.body.WhatsApp }}`  
     - `form`: `={{ $json.body.form_id }}`  
     - `email_valid` (boolean): `={{ $json.body['E-mail'].isEmail() }}`

3. **Add Mautic Node for Contact Creation**  
   - Add a **Mautic** node named `CreateContactMautic`.  
   - Connect input from `LeadData`.  
   - Operation: Create Contact.  
   - Map fields:  
     - Email: `{{$json.email}}`  
     - First Name: `{{$json.name}}`  
     - Additional Fields > Mobile: `{{$json.mobile}}`  
   - Set credentials to your Mautic account credentials.

4. **Add If Node for Email Validation Check**  
   - Add an **If** node named `CheckEmailValid`.  
   - Connect input from `CreateContactMautic`.  
   - Condition: Check if `email_valid` from `LeadData` is `true`.  
     - Expression: `={{ $('LeadData').item.json.email_valid }}` equals `true`.

5. **Add Mautic Node to Mark Contacts as Do Not Contact**  
   - Add a **Mautic** node named `LeadMauticDNC`.  
   - Connect it to the **False** branch of `CheckEmailValid`.  
   - Operation: Edit Do Not Contact List.  
   - Set Contact ID: `{{$json.id}}` (from contact creation response).  
   - Reason: Use code `3` (meaning invalid email).  
   - Comments: "Did not pass basic email validation."  
   - Use the same Mautic credentials as before.

6. **Add No Operation (NoOp) Node for Workflow End**  
   - Add a **NoOp** node named `End`.  
   - Connect input from `LeadMauticDNC`.  
   - This node marks the end of the workflow.

7. **Connect Workflow Paths**  
   - Connect `WordpressForm` → `LeadData` → `CreateContactMautic` → `CheckEmailValid`.  
   - From `CheckEmailValid`:  
     - **True** branch: no further node (workflow ends).  
     - **False** branch: connect to `LeadMauticDNC` → `End`.

8. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add sticky notes near the logical blocks explaining their purpose, using the content as in the original workflow.

9. **Credentials Setup**  
   - Setup Mautic API credentials with required authorization (OAuth2 or API key).  
   - Ensure the webhook URL is accessible from Wordpress and configure the Wordpress form to submit data to this URL.

10. **Testing and Deployment**  
    - Test using the webhook URL in a development environment.  
    - Once verified, switch to production webhook URL and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| "N8N offers powerful expression extensions to format and validate data received by workflows."            | Useful for data normalization and validation in automation workflows.                                       |
| "Mautic API integration requires properly configured credentials; ensure permissions for contact creation and list editing." | Credential setup note for Mautic integration.                                                                |
| "Wordpress forms can be built with Elementor, WPForms, or similar, and should be configured to POST data to the n8n webhook URL." | Wordpress form setup guidance.                                                                               |
| "For more detailed Mautic API references, visit: https://developer.mautic.org/"                            | Official Mautic developer documentation.                                                                     |

---

This structured document enables clear understanding, reproduction, and modification of the workflow by technical users and automation systems alike.