Generate and Send Contract Documents with Typeform, Google Docs and Gmail

https://n8nworkflows.xyz/workflows/generate-and-send-contract-documents-with-typeform--google-docs-and-gmail-6817


# Generate and Send Contract Documents with Typeform, Google Docs and Gmail

### 1. Workflow Overview

This workflow automates the generation and delivery of contract documents based on client information submitted through a Typeform form. It is designed for businesses that need to efficiently create personalized contract documents and send them via email without manual intervention.

Logical blocks included:

- **1.1 Input Reception**: Captures client inputs from Typeform submissions.
- **1.2 Data Preparation**: Extracts and organizes relevant data from the form response.
- **1.3 Document Generation**: Uses Google Docs templates to create a customized contract document.
- **1.4 Document Export**: Converts the generated Google Doc into a PDF file.
- **1.5 Email Delivery**: Sends the PDF contract as an email attachment to the client.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for new submissions on a specified Typeform form and triggers the workflow.
- **Nodes Involved:** `Typeform Trigger`
- **Node Details:**

  - **Type:** Typeform Trigger  
  - **Technical Role:** Entry point that activates the workflow when a form is submitted.  
  - **Configuration:**  
    - `formId`: Unique identifier of the Typeform form to monitor (replace `"your-form-id"` with actual ID).  
    - Credentials: Uses `Typeform API` credentials for authentication.  
  - **Key Expressions:** None (trigger-based).  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connected to `Set Variables`.  
  - **Version Requirements:** Compatible with n8n version supporting Typeform Trigger node (version 1 used).  
  - **Potential Failures:**  
    - Invalid or expired API credentials causing authentication errors.  
    - Incorrect form ID leading to no triggers.  
    - Typeform service downtime or network issues causing missed triggers.  
  - **Sub-workflows:** None.

#### 1.2 Data Preparation

- **Overview:** Extracts relevant answers from the Typeform submission and assigns them to workflow variables for use in later nodes.
- **Nodes Involved:** `Set Variables`
- **Node Details:**

  - **Type:** Set  
  - **Technical Role:** Maps and stores specific form answers into named variables.  
  - **Configuration:**  
    - Extracts fields from the JSON response:  
      - `client_name` from first answer's text.  
      - `project_scope` from second answer's text.  
      - `deadline` from third answer’s text.  
      - `email` from fourth answer’s email field.  
  - **Key Expressions:**  
    - `={{$json["answers"][0].text}}` for client name.  
    - `={{$json["answers"][1].text}}` for project scope.  
    - `={{$json["answers"][2].text}}` for deadline.  
    - `={{$json["answers"][3].email}}` for client email.  
  - **Input Connections:** Receives from `Typeform Trigger`.  
  - **Output Connections:** Leads to `Fill Contract Template`.  
  - **Version Requirements:** Compatible with n8n Set node version 1.  
  - **Potential Failures:**  
    - Index errors if form answers are missing or reordered.  
    - Missing or malformed email causing delivery issues downstream.  
  - **Sub-workflows:** None.

#### 1.3 Document Generation

- **Overview:** Fills a Google Docs template with the extracted client data to generate a personalized contract.
- **Nodes Involved:** `Fill Contract Template`
- **Node Details:**

  - **Type:** Google Docs  
  - **Technical Role:** Performs template replacement in a Google Docs document using mapped variables.  
  - **Configuration:**  
    - Mode set to `template` (replaces placeholders with actual data).  
    - Fields:  
      - `{{client_name}}` → value from `client_name` variable.  
      - `{{project_scope}}` → value from `project_scope`.  
      - `{{deadline}}` → value from `deadline`.  
    - `documentId`: Google Docs template ID (replace `"your-template-id"` with actual template ID).  
    - Credentials: Uses `Google API` credentials with sufficient permissions.  
  - **Key Expressions:**  
    - `={{$json["client_name"]}}` etc. for field values.  
  - **Input Connections:** Receives from `Set Variables`.  
  - **Output Connections:** Passes to `Export as PDF`.  
  - **Version Requirements:** Requires Google Docs node version 1 or higher.  
  - **Potential Failures:**  
    - Missing or incorrect document ID causing template errors.  
    - Insufficient Google API permissions.  
    - Placeholders in the template not matching the field names.  
  - **Sub-workflows:** None.

#### 1.4 Document Export

- **Overview:** Converts the filled Google Docs document into a PDF format for distribution.
- **Nodes Involved:** `Export as PDF`
- **Node Details:**

  - **Type:** Google Drive  
  - **Technical Role:** Exports Google Docs document as a PDF file.  
  - **Configuration:**  
    - Operation: `export`.  
    - `fileId`: Dynamically retrieved from the output of `Fill Contract Template` node’s `documentId` field.  
    - MIME type set to `application/pdf`.  
    - Credentials: Uses the same `Google API` credentials.  
  - **Key Expressions:**  
    - `={{$node["Fill Contract Template"].json["documentId"]}}` for the file ID to export.  
  - **Input Connections:** Receives from `Fill Contract Template`.  
  - **Output Connections:** Connected to `Send via Email`.  
  - **Version Requirements:** Requires Google Drive node version 1 or higher.  
  - **Potential Failures:**  
    - File ID not found or invalid causing export failure.  
    - API permission issues preventing export or download.  
    - Network or quota limits on Google Drive API.  
  - **Sub-workflows:** None.

#### 1.5 Email Delivery

- **Overview:** Sends the exported PDF contract as an email attachment to the client’s email address.
- **Nodes Involved:** `Send via Email`
- **Node Details:**

  - **Type:** Gmail  
  - **Technical Role:** Sends an email with a PDF attachment to the client.  
  - **Configuration:**  
    - Email body includes personalized greeting using `client_name`.  
    - Subject: "Your Contract".  
    - Recipient email dynamically set from the `email` variable.  
    - Sender email hardcoded (replace `"your-email@example.com"` with actual sender).  
    - Attaches the PDF file as a binary property named `"data"`.  
    - Credentials: Uses OAuth2 credentials for Gmail (`Gmail OAuth`).  
  - **Key Expressions:**  
    - `Hi {{$json["client_name"]}},` in email text.  
    - `={{$json["email"]}}` for recipient.  
  - **Input Connections:** Receives from `Export as PDF`.  
  - **Output Connections:** Terminal node, no outputs.  
  - **Version Requirements:** Gmail node version 1 or higher with OAuth2 support.  
  - **Potential Failures:**  
    - Authentication failures due to expired or invalid OAuth tokens.  
    - Invalid recipient email format.  
    - Attachment size limits exceeded.  
    - Gmail API quota limits or temporary outages.  
  - **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                  | Input Node(s)       | Output Node(s)     | Sticky Note                                               |
|---------------------|--------------------|--------------------------------|---------------------|--------------------|-----------------------------------------------------------|
| Typeform Trigger    | Typeform Trigger   | Entry trigger for form submits | None                | Set Variables      |                                                           |
| Set Variables       | Set                | Extracts and sets form answers | Typeform Trigger    | Fill Contract Template |                                                           |
| Fill Contract Template | Google Docs       | Fills contract template         | Set Variables       | Export as PDF      |                                                           |
| Export as PDF       | Google Drive       | Exports Google Doc as PDF       | Fill Contract Template | Send via Email    |                                                           |
| Send via Email      | Gmail              | Sends contract email with PDF   | Export as PDF       | None               |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node: Typeform Trigger**  
   - Add a Typeform Trigger node.  
   - Set `formId` to your Typeform form’s ID.  
   - Configure credentials with your Typeform API key.  
   - This node will listen for new form submissions to start the workflow.

2. **Add the Set Variables Node:**  
   - Create a Set node connected to the Typeform Trigger output.  
   - Add variables:  
     - `client_name` → Expression: `{{$json["answers"][0].text}}`  
     - `project_scope` → Expression: `{{$json["answers"][1].text}}`  
     - `deadline` → Expression: `{{$json["answers"][2].text}}`  
     - `email` → Expression: `{{$json["answers"][3].email}}`  
   - This maps the form answers to named fields for reuse.

3. **Insert the Google Docs Node:**  
   - Add a Google Docs node connected to the Set node.  
   - Set mode to `template`.  
   - Add fields mapping placeholders to variables:  
     - `{{client_name}}` → `{{$json["client_name"]}}`  
     - `{{project_scope}}` → `{{$json["project_scope"]}}`  
     - `{{deadline}}` → `{{$json["deadline"]}}`  
   - Enter the Google Docs template ID of your contract document.  
   - Configure Google API credentials with permissions to edit Docs.

4. **Add the Google Drive Export Node:**  
   - Add a Google Drive node connected to the Google Docs node.  
   - Set operation to `export`.  
   - Set `fileId` to expression: `{{$node["Fill Contract Template"].json["documentId"]}}`.  
   - Set MIME type to `application/pdf`.  
   - Use the same Google API credentials.

5. **Configure the Gmail Node for Sending Email:**  
   - Add a Gmail node connected to the Google Drive node.  
   - Set the email subject to `"Your Contract"`.  
   - Email body:  
     ```
     Hi {{$json["client_name"]}},
     
     Please find attached your contract.
     
     Regards,
     Your Company
     ```  
   - Set recipient email to expression: `{{$json["email"]}}`.  
   - Set sender email to your verified Gmail address.  
   - Attach the PDF binary data with property name `"data"`.  
   - Configure Gmail OAuth2 credentials with send email scope.

6. **Verify Connections and Credentials:**  
   - Ensure all nodes are properly connected in the sequence:  
     Typeform Trigger → Set Variables → Fill Contract Template → Export as PDF → Send via Email.  
   - Double-check all credentials are active and authorized.

7. **Test the Workflow:**  
   - Submit a test entry in your Typeform form.  
   - Confirm the contract is generated, exported as PDF, and emailed to the specified address.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| Replace placeholder IDs (`your-form-id`, `your-template-id`) and emails with actual values.   | Workflow configuration                            |
| Google API credentials require enabling Docs and Drive APIs with appropriate scopes.           | Google Cloud Console setup                        |
| Gmail OAuth credentials require enabling Gmail API and configuring OAuth consent screen.       | Google Cloud Console OAuth setup                  |
| Ensure the Typeform form's answer order matches the index assumptions in the Set node.         | Typeform form design                              |
| For large attachments, check Gmail attachment size limits (typically 25 MB).                   | Gmail API documentation                           |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly conforms to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.