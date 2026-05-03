Streamlining Document Automation with n8n and JSReport

https://n8nworkflows.xyz/workflows/streamlining-document-automation-with-n8n-and-jsreport-2254


# Streamlining Document Automation with n8n and JSReport

---

### 1. Workflow Overview

This n8n workflow automates the generation and emailing of PDF or Word documents using JSReport, targeting teams in accounting, human resources, and IT project management. It enables users to input document data via a form (or alternatively via webhooks from external systems), generate customized documents through JSReport templates, and send the resulting files by email.

**Logical blocks:**

- **1.1 Input Reception:** Captures user input data through an n8n form trigger node.
- **1.2 Document Generation via JSReport:** Sends a structured JSON payload to JSReport’s API to generate a PDF or Word document based on predefined templates.
- **1.3 Email Dispatch:** Sends the generated document as an email attachment using Gmail integration.
- **1.4 Documentation & Guidance:** Sticky notes providing workflow context, usage instructions, and API information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block collects invoice or document data from a user via an n8n form interface. It serves as the entry point for the workflow and gathers all necessary fields to populate the JSReport template.

- **Nodes Involved:**  
  - `Form Invoice`

- **Node Details:**  
  - **Form Invoice**  
    - *Type & Role:* `formTrigger` node; initiates workflow on form submission.  
    - *Configuration:*  
      - Form titled "Create Facture" with fields for buyer’s name, address, country, and two line items (names and prices).  
      - Required fields: buyer name, road, and country.  
      - Numeric validation for item prices where applicable.  
      - Webhook path specified to receive form submissions.  
    - *Expressions / Variables:* The form data is accessed downstream via `$json` with keys matching the form fields (e.g., `buyer name`, `Item 1 Price`).  
    - *Connections:* Output main connection links to `Get PDF From JSReport` node.  
    - *Version Requirements:* Uses formTrigger v2; ensure n8n version >=1.38.2 for compatibility.  
    - *Potential Failure Modes:*  
      - Missing required fields (prevented by form validation).  
      - Malformed or unexpected input data causing template errors downstream.  
      - Webhook connectivity issues or misconfiguration.  
    - *Sub-workflow:* None.

#### 2.2 Document Generation via JSReport

- **Overview:**  
  This block assembles a JSON request payload using form data and sends it to JSReport’s API to generate the desired document (PDF/Word). It uses HTTP Basic Authentication and invokes a predefined JSReport template (`invoice-main`).

- **Nodes Involved:**  
  - `Get PDF From JSReport`

- **Node Details:**  
  - **Get PDF From JSReport**  
    - *Type & Role:* `httpRequest` node; performs POST request to JSReport API to generate document.  
    - *Configuration:*  
      - URL: `https://xxx.jsreportonline.net/api/report` (replace `xxx` with your JSReport account domain).  
      - Method: POST.  
      - Body: JSON structured with `template.name` set to `invoice-main`.  
      - `data` field dynamically populated with form inputs, accessed via expressions like `{{ $json["buyer name"] }}` for buyer info and items.  
      - Authentication: HTTP Basic Auth with credentials stored securely in n8n.  
      - Sends JSON payload (`specifyBody` as JSON).  
    - *Expressions / Variables:* Uses templated expressions to inject form data into the JSReport payload.  
    - *Input / Output:* Receives input from `Form Invoice`, outputs the generated document binary data and metadata to `Send invoice`.  
    - *Version Requirements:* Node version 4.2 or higher recommended for expression and authentication support.  
    - *Potential Failure Modes:*  
      - Authentication failures due to incorrect credentials.  
      - JSReport API errors if the template name is incorrect or if data does not match the template schema.  
      - Network issues or timeouts.  
      - Malformed JSON due to invalid input data or expression errors.  
    - *Sub-workflow:* None.

#### 2.3 Email Dispatch

- **Overview:**  
  This block sends the generated document as an email attachment using Gmail OAuth2 authentication.

- **Nodes Involved:**  
  - `Send invoice`

- **Node Details:**  
  - **Send invoice**  
    - *Type & Role:* `gmail` node; sends email with attachment.  
    - *Configuration:*  
      - Recipient email fixed to `contact@nonocode.fr`.  
      - Subject: "New Facture".  
      - Message body includes a polite notification about the attached invoice.  
      - Attachment is the binary output from the previous node (`Get PDF From JSReport`).  
      - Uses Gmail OAuth2 credentials stored in n8n.  
    - *Expressions / Variables:* No dynamic content in message or subject; could be customized by users.  
    - *Input / Output:* Takes binary document from prior node; no further output.  
    - *Version Requirements:* Gmail node version 2.1 or higher recommended.  
    - *Potential Failure Modes:*  
      - Authentication errors if OAuth2 token is expired or revoked.  
      - Email sending failures due to invalid recipient or connectivity issues.  
      - Attachment size limits or format incompatibilities.  
    - *Sub-workflow:* None.

#### 2.4 Documentation & Guidance

- **Overview:**  
  Sticky notes provide explanations and usage instructions to help users understand the workflow’s purpose and how to configure it.

- **Nodes Involved:**  
  - `Sticky Note`  
  - `Sticky Note1`

- **Node Details:**  
  - **Sticky Note**  
    - *Type & Role:* `stickyNote` node; informative.  
    - *Content:* Describes the overall billing automation process from form input to document creation and email sending.  
  - **Sticky Note1**  
    - *Type & Role:* `stickyNote` node; informative.  
    - *Content:* Contains detailed JSReport API URL, usage instructions, and links to create accounts and access API docs.  
    - *Links:*  
      - JSReport online signup: https://jsreport.net/online  
      - JSReport API docs: https://jsreport.net/learn/api  
  - *Connections:* None; these are informational and do not affect workflow execution.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                      | Input Node(s)      | Output Node(s)        | Sticky Note                                                                                                              |
|---------------------|----------------------|------------------------------------|--------------------|-----------------------|--------------------------------------------------------------------------------------------------------------------------|
| Form Invoice        | formTrigger          | Input Reception (form data capture) | -                  | Get PDF From JSReport  | Allows you to enter invoice information                                                                                   |
| Get PDF From JSReport| httpRequest          | Document Generation (JSReport API)  | Form Invoice       | Send invoice          | Generating the document in JSReport                                                                                        |
| Send invoice        | gmail                | Email Dispatch (send PDF attachment)| Get PDF From JSReport| -                      | Using GMAIL to send the invoice                                                                                            |
| Sticky Note         | stickyNote           | Documentation & Guidance            | -                  | -                     | Describes billing automation process from form input to document creation and email sending                               |
| Sticky Note1        | stickyNote           | Documentation & Guidance            | -                  | -                     | JSReport API URL and usage instructions. Links: https://jsreport.net/online, https://jsreport.net/learn/api               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a `formTrigger` node named `Form Invoice`.  
   - Configure the form:  
     - Title: "Create Facture"  
     - Description: "Create a PDF invoice from an n8n and JSReport form"  
     - Fields:  
       - buyer name (required, text)  
       - buyer road (required, text)  
       - buyer country (required, text)  
       - Item 1 Name (text)  
       - Item 1 Price (number)  
       - Item 2 Name (text)  
       - Item 2 Price (number)  
   - Save and note the webhook path generated (e.g., `1d0c5777-4033-4bf4-8d0e-8a2069d79c86`).

2. **Create the HTTP Request Node to Call JSReport**  
   - Add an `httpRequest` node named `Get PDF From JSReport`.  
   - Configure:  
     - URL: `https://xxx.jsreportonline.net/api/report` (replace `xxx` with your JSReport subdomain).  
     - Method: POST  
     - Authentication: HTTP Basic Auth — create or select credentials with your JSReport API username and password.  
     - Body Content Type: JSON  
     - Body: Use the following JSON with expressions to map form data:  
       ```json
       {
         "template": { "name": "invoice-main" },
         "data": {
           "number": "123",
           "seller": {
             "name": "Next Step Webs, Inc.",
             "road": "12345 Sunny Road",
             "country": "Sunnyville, TX 12345"
           },
           "buyer": {
             "name": "{{ $json[\"buyer name\"] }}",
             "road": "{{ $json[\"buyer road\"] }}",
             "country": "{{ $json[\"buyer country\"] }}"
           },
           "items": [
             {
               "name": "{{ $json[\"Item 1 Name\"] }}",
               "price": {{ $json[\"Item 1 Price\"] }}
             },
             {
               "name": "{{ $json[\"Item 2 Name\"] }}",
               "price": {{ $json[\"Item 2 Price\"] }}
             }
           ]
         }
       }
       ```
     - Enable `Send Body` and specify body as JSON.

3. **Create the Gmail Node to Send the Document**  
   - Add a `gmail` node named `Send invoice`.  
   - Configure:  
     - OAuth2 credentials: select or create Gmail OAuth2 credentials.  
     - Recipient email: `contact@nonocode.fr` (customize as needed).  
     - Subject: "New Facture"  
     - Message Body:  
       ```
       Good morning,  
       
       Please find your invoice.  
       
       Sincerely,
       ```  
     - Attachments: Add the binary output from the `Get PDF From JSReport` node (ensure binary property is forwarded).  
   - Save the node.

4. **Link the Nodes**  
   - Connect `Form Invoice` main output to `Get PDF From JSReport` input.  
   - Connect `Get PDF From JSReport` main output to `Send invoice` input.

5. **Create Sticky Notes (Optional for Documentation)**  
   - Add `stickyNote` nodes with content describing the workflow purpose and instructions, including links to JSReport and example usage.

6. **Credentials Setup**  
   - Set up HTTP Basic Auth credentials for JSReport with your JSReport username and password.  
   - Set up Gmail OAuth2 credentials with appropriate scopes for sending emails.

7. **Testing and Deployment**  
   - Activate the workflow.  
   - Submit the form or send data to the webhook to trigger the workflow.  
   - Verify receipt of the email with the attached PDF generated by JSReport.

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                              |
|---------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| JSReport online FREE account creation and API usage guide.                                                         | https://jsreport.net/online and https://jsreport.net/learn/api |
| Video demonstration for JSReport integration with n8n.                                                             | https://youtu.be/nz1SKdOKAhM?si=aMWGMgMuYCnky28z             |
| Video overview of this n8n workflow template.                                                                       | https://youtu.be/jjjXCj3flPI?si=C73gd51ZeQ_l-1Bh             |
| Detailed setup guidance video for configuring HTTP Request with JSReport.                                           | https://youtu.be/fRoRze7CBY4                                  |
| Workflow created and tested on n8n v1.38.2; ensure compatibility with this version or later.                        | n8n version requirement                                       |

---

This documentation enables users and developers to understand, reproduce, and extend the workflow for automated document generation and emailing with JSReport and n8n. It highlights configuration details, potential error points, and integration notes critical for smooth operation.