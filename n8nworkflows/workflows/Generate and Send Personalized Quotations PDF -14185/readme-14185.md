Generate and Send Personalized Quotations PDF 

https://n8nworkflows.xyz/workflows/generate-and-send-personalized-quotations-pdf--14185


# Generate and Send Personalized Quotations PDF 

# 1. Workflow Overview

This workflow generates a personalized quotation PDF from a submitted form, enriches the content with AI-generated business language, and emails the final document to the client.

It is designed for agencies, consultants, or service providers who want to automate quotation generation while keeping the result personalized and branded. The workflow combines form intake, AI text generation, Google Docs templating, PDF export, and Gmail delivery.

## 1.1 Input Reception

The workflow starts with an n8n Form Trigger that collects quotation request details such as client identity, service type, pricing, and optional notes.

## 1.2 AI Personalization

After the form is submitted, an OpenAI node generates two structured content fields in JSON format:

- a personalized introduction
- a scope-of-work summary

These outputs are later injected into the quotation document and email body.

## 1.3 Document Creation from Template

The workflow duplicates a predefined Google Docs quotation template stored in Google Drive. The copied file is renamed dynamically using the company name and service name from the form submission.

## 1.4 Template Variable Replacement

The copied Google Doc is updated by replacing template placeholders such as `{{client_name}}`, `{{price}}`, and `{{intro}}` with live form data and AI-generated text.

## 1.5 PDF Export

The completed Google Doc is exported as a PDF file using Google Drive’s file conversion capability.

## 1.6 Email Delivery

The generated PDF is attached to an outgoing Gmail message sent to the client. The email body also includes the personalized introduction, scope summary, total price, and validity date.

---

# 2. Block-by-Block Analysis

## Block 1 — Form Intake

### Overview

This block provides the entry point for the workflow. It exposes a hosted form where a user enters quotation details, and the submitted data becomes the source for all downstream processing.

### Nodes Involved

- Quotation Form

### Node Details

#### Quotation Form

- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that waits for a form submission and emits one item containing the submitted field values.

- **Configuration choices:**
  - Form title: `Quotation Request`
  - Form description: `Fill in the details below to generate a personalized quotation.`
  - Response mode: `lastNode`, meaning the form response depends on the final node completing
  - Fields collected:
    - Client Name — required
    - Client Email — required
    - Company Name — required
    - Service — required
    - Service Description — required
    - Price — required
    - Valid Until — required
    - Notes — optional

- **Key expressions or variables used:**  
  This node does not reference earlier nodes. Its output fields are later used extensively via expressions like:
  - `$json['Client Name']`
  - `$json['Company Name']`
  - `$json['Service Description']`

- **Input and output connections:**
  - Input: none
  - Output: `OpenAI Personalize`

- **Version-specific requirements:**  
  Uses type version `2.2`. Form Trigger behavior and field UI depend on your n8n version supporting this node version or equivalent.

- **Edge cases or potential failure types:**
  - Invalid or mistyped email is not formally validated here beyond being a string field
  - Optional `Notes` may be empty, which can affect prompt quality but not execution
  - Because response mode is `lastNode`, long downstream execution may delay the form response
  - If later nodes fail, the form submitter may receive an error rather than a success response

- **Sub-workflow reference:**  
  None

---

## Block 2 — AI Personalization

### Overview

This block transforms raw form details into polished business language suitable for a quotation. It instructs OpenAI to return strictly structured JSON so that later nodes can reliably consume the generated content.

### Nodes Involved

- OpenAI Personalize

### Node Details

#### OpenAI Personalize

- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Calls OpenAI’s chat model to generate custom quotation text.

- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.7`
  - System message defines the assistant as a professional business proposal writer for Baraco AI
  - User message requests:
    - a personalized introduction paragraph
    - a brief scope-of-work summary
  - `jsonOutput: true` is enabled so the response is expected as JSON

- **Key expressions or variables used:**
  - `{{ $json['Client Name'] }}`
  - `{{ $json['Company Name'] }}`
  - `{{ $json['Service'] }}`
  - `{{ $json['Service Description'] }}`
  - `{{ $json['Notes'] }}`

  The prompt explicitly asks the model to respond only with:
  - `intro`
  - `scope_summary`

  Later references assume the node output is available at:
  - `$('OpenAI Personalize').item.json.message.content.intro`
  - `$('OpenAI Personalize').item.json.message.content.scope_summary`

- **Input and output connections:**
  - Input: `Quotation Form`
  - Output: `Copy Template`

- **Version-specific requirements:**  
  Uses type version `1.8`. Output structure may vary slightly across n8n/OpenAI node versions, especially when JSON output mode changes.

- **Edge cases or potential failure types:**
  - OpenAI authentication failure or quota exhaustion
  - Model availability issues
  - The model may return malformed JSON despite the instruction, causing downstream expression failures
  - Empty `Notes` may produce less contextual output
  - If output format differs from expected `message.content.intro`, later replace/email expressions may break

- **Sub-workflow reference:**  
  None

---

## Block 3 — Document Template Duplication

### Overview

This block creates a working copy of a master Google Docs quotation template. It ensures the original template remains unchanged while each request gets a uniquely named document.

### Nodes Involved

- Copy Template

### Node Details

#### Copy Template

- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Duplicates a Google Drive file, in this case a Google Docs template.

- **Configuration choices:**
  - Operation: `copy`
  - Source file ID: hardcoded placeholder `REPLACE_WITH_TEMPLATE_DOC_ID`
  - New file name built from form values:
    - `Quotation - {{ Company Name }} - {{ Service }}`

- **Key expressions or variables used:**
  - `$('Quotation Form').item.json['Company Name']`
  - `$('Quotation Form').item.json['Service']`

- **Input and output connections:**
  - Input: `OpenAI Personalize`
  - Output: `Replace Variables`

- **Version-specific requirements:**  
  Uses Google Drive node version `3`.

- **Edge cases or potential failure types:**
  - The template file ID must be replaced with a valid Google Docs file ID
  - Google Drive OAuth scope must permit copying files
  - If the authenticated account cannot access the template, the node fails
  - Invalid characters in the generated file name are usually handled by Google Drive, but naming may be inconsistent
  - If the template is not a Google Docs file, later Google Docs replacement may fail

- **Sub-workflow reference:**  
  None

---

## Block 4 — Variable Injection into Google Docs

### Overview

This block fills the copied quotation document with live data. It replaces all template placeholders using both submitted form values and AI-generated content.

### Nodes Involved

- Replace Variables

### Node Details

#### Replace Variables

- **Type and technical role:** `n8n-nodes-base.googleDocs`  
  Updates a Google Docs file by applying multiple replace-all actions.

- **Configuration choices:**
  - Operation: `update`
  - Replace actions for these placeholders:
    - `{{client_name}}`
    - `{{company_name}}`
    - `{{client_email}}`
    - `{{service}}`
    - `{{price}}`
    - `{{valid_until}}`
    - `{{notes}}`
    - `{{intro}}`
    - `{{scope_summary}}`

- **Key expressions or variables used:**
  - Form-driven values:
    - `$('Quotation Form').item.json['Client Name']`
    - `$('Quotation Form').item.json['Company Name']`
    - `$('Quotation Form').item.json['Client Email']`
    - `$('Quotation Form').item.json['Service']`
    - `$('Quotation Form').item.json['Price']`
    - `$('Quotation Form').item.json['Valid Until']`
    - `$('Quotation Form').item.json['Notes']`
  - AI-driven values:
    - `$('OpenAI Personalize').item.json.message.content.intro`
    - `$('OpenAI Personalize').item.json.message.content.scope_summary`

- **Input and output connections:**
  - Input: `Copy Template`
  - Output: `Export as PDF`

- **Version-specific requirements:**  
  Uses Google Docs node version `2`. The node must support replace-all update actions.

- **Edge cases or potential failure types:**
  - The Google Docs credential name shown is `Google Sheets account 2`, which may be only a label, but the credential itself must actually be a valid Google Docs OAuth credential
  - The node must know which document to update; this workflow implies the copied Google Doc is passed from the previous node. If the Google Docs node version/configuration requires an explicit document ID and it is not set, execution may fail or depend on implicit input behavior
  - If placeholders are absent from the template, replacement silently has no visible effect
  - If `Notes` is empty, the `{{notes}}` replacement may become blank
  - If AI output path is wrong or malformed, replacement expressions will fail
  - Multi-line content in `scope_summary` may not render exactly as intended depending on Google Docs formatting

- **Sub-workflow reference:**  
  None

---

## Block 5 — PDF Export

### Overview

This block converts the finished Google Doc into a PDF binary file. The resulting binary is passed forward for email attachment.

### Nodes Involved

- Export as PDF

### Node Details

#### Export as PDF

- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads a Google Drive file and converts Google Docs content into PDF format during download.

- **Configuration choices:**
  - Operation: `download`
  - File ID comes from the copied document:
    - `$('Copy Template').item.json.id`
  - Google file conversion:
    - Docs to format: `application/pdf`

- **Key expressions or variables used:**
  - `={{ $('Copy Template').item.json.id }}`

- **Input and output connections:**
  - Input: `Replace Variables`
  - Output: `Send Quotation Email`

- **Version-specific requirements:**  
  Uses Google Drive node version `3`.

- **Edge cases or potential failure types:**
  - If the previous document update has not completed properly, the PDF may reflect incomplete content
  - If the file ID is missing or incorrect, export fails
  - Permission issues on the copied document can block download
  - Large or complex documents may increase processing time
  - Binary output depends on workflow binary mode; this workflow uses `binaryMode: separate`

- **Sub-workflow reference:**  
  None

---

## Block 6 — Email Delivery

### Overview

This final block sends the finished quotation to the client by Gmail, attaching the generated PDF. It also includes the same personalized AI-generated content in the email body.

### Nodes Involved

- Send Quotation Email

### Node Details

#### Send Quotation Email

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an outbound email through Gmail with a binary attachment.

- **Configuration choices:**
  - Recipient:
    - `$('Quotation Form').item.json['Client Email']`
  - Subject:
    - `Quotation from Baraco AI - {{ Service }}`
  - Body includes:
    - greeting by client name
    - AI-generated intro
    - mention of attached quotation
    - scope of work
    - total price
    - validity date
    - closing signature as Baraco AI
  - Attachments enabled with `attachmentsBinary`

- **Key expressions or variables used:**
  - `$('Quotation Form').item.json['Client Email']`
  - `$('Quotation Form').item.json['Client Name']`
  - `$('Quotation Form').item.json['Price']`
  - `$('Quotation Form').item.json['Valid Until']`
  - `$('Quotation Form').item.json['Service']`
  - `$('OpenAI Personalize').item.json.message.content.intro`
  - `$('OpenAI Personalize').item.json.message.content.scope_summary`

- **Input and output connections:**
  - Input: `Export as PDF`
  - Output: none

- **Version-specific requirements:**  
  Uses Gmail node version `2.1`.

- **Edge cases or potential failure types:**
  - Gmail OAuth must support send permissions
  - If attachment binary property is not correctly present from the previous node, the email may send without attachment or fail
  - Recipient email address is only a free-text form input; malformed addresses may trigger send errors
  - Long AI-generated text may make the email body less polished if not constrained
  - Gmail account sending limits may apply

- **Sub-workflow reference:**  
  None

---

## Block 7 — Documentation Sticky Note

### Overview

This node provides inline operational documentation inside the workflow canvas. It is not executed but contains essential implementation notes, required placeholders, and credential guidance.

### Nodes Involved

- Sticky Note

### Node Details

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Non-executable annotation node used for visual documentation.

- **Configuration choices:**
  - Contains workflow summary, flow order, setup requirements, and credential requirements

- **Key expressions or variables used:**  
  None

- **Input and output connections:**
  - Input: none
  - Output: none

- **Version-specific requirements:**  
  Uses sticky note version `1`.

- **Edge cases or potential failure types:**  
  None during execution, since it does not run.

- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Quotation Form | n8n-nodes-base.formTrigger | Receives quotation request data from a hosted form |  | OpenAI Personalize | ## Quotation Generator PDF<br>Generates personalized quotations from a form submission, creates a branded PDF, and emails it to the client.<br>### Flow<br>Form > OpenAI Personalize > Copy Template > Replace Variables > Export PDF > Send Email<br>### How It Works<br>1. Client fills out the quotation form (name, email, company, service, price, etc.)<br>2. OpenAI generates a personalized intro paragraph and scope summary<br>3. Copies the Google Docs template to a new document<br>4. Replaces all {{variables}} in the copied doc with form data + AI content<br>5. Exports the document as PDF<br>6. Sends the PDF as an email attachment via Gmail<br>### Setup Required<br>1. Create a Google Docs template with these variables:<br>{{client_name}}, {{company_name}}, {{client_email}}, {{service}}, {{price}}, {{valid_until}}, {{notes}}, {{intro}}, {{scope_summary}}<br>2. Copy the template Document ID and paste it in the Copy Template node<br>3. Configure Google Docs OAuth credential in Replace Variables node<br>4. Configure Gmail OAuth credential in Send Quotation Email node<br>### Credentials Required<br>- OpenAI API key<br>- Google Drive OAuth<br>- Google Docs OAuth<br>- Gmail OAuth |
| OpenAI Personalize | @n8n/n8n-nodes-langchain.openAi | Generates personalized introduction and scope summary in JSON | Quotation Form | Copy Template | ## Quotation Generator PDF<br>Generates personalized quotations from a form submission, creates a branded PDF, and emails it to the client.<br>### Flow<br>Form > OpenAI Personalize > Copy Template > Replace Variables > Export PDF > Send Email<br>### How It Works<br>1. Client fills out the quotation form (name, email, company, service, price, etc.)<br>2. OpenAI generates a personalized intro paragraph and scope summary<br>3. Copies the Google Docs template to a new document<br>4. Replaces all {{variables}} in the copied doc with form data + AI content<br>5. Exports the document as PDF<br>6. Sends the PDF as an email attachment via Gmail<br>### Setup Required<br>1. Create a Google Docs template with these variables:<br>{{client_name}}, {{company_name}}, {{client_email}}, {{service}}, {{price}}, {{valid_until}}, {{notes}}, {{intro}}, {{scope_summary}}<br>2. Copy the template Document ID and paste it in the Copy Template node<br>3. Configure Google Docs OAuth credential in Replace Variables node<br>4. Configure Gmail OAuth credential in Send Quotation Email node<br>### Credentials Required<br>- OpenAI API key<br>- Google Drive OAuth<br>- Google Docs OAuth<br>- Gmail OAuth |
| Copy Template | n8n-nodes-base.googleDrive | Creates a copy of the master Google Docs quotation template | OpenAI Personalize | Replace Variables | ## Quotation Generator PDF<br>Generates personalized quotations from a form submission, creates a branded PDF, and emails it to the client.<br>### Flow<br>Form > OpenAI Personalize > Copy Template > Replace Variables > Export PDF > Send Email<br>### How It Works<br>1. Client fills out the quotation form (name, email, company, service, price, etc.)<br>2. OpenAI generates a personalized intro paragraph and scope summary<br>3. Copies the Google Docs template to a new document<br>4. Replaces all {{variables}} in the copied doc with form data + AI content<br>5. Exports the document as PDF<br>6. Sends the PDF as an email attachment via Gmail<br>### Setup Required<br>1. Create a Google Docs template with these variables:<br>{{client_name}}, {{company_name}}, {{client_email}}, {{service}}, {{price}}, {{valid_until}}, {{notes}}, {{intro}}, {{scope_summary}}<br>2. Copy the template Document ID and paste it in the Copy Template node<br>3. Configure Google Docs OAuth credential in Replace Variables node<br>4. Configure Gmail OAuth credential in Send Quotation Email node<br>### Credentials Required<br>- OpenAI API key<br>- Google Drive OAuth<br>- Google Docs OAuth<br>- Gmail OAuth |
| Replace Variables | n8n-nodes-base.googleDocs | Replaces template placeholders in the copied Google Doc | Copy Template | Export as PDF | ## Quotation Generator PDF<br>Generates personalized quotations from a form submission, creates a branded PDF, and emails it to the client.<br>### Flow<br>Form > OpenAI Personalize > Copy Template > Replace Variables > Export PDF > Send Email<br>### How It Works<br>1. Client fills out the quotation form (name, email, company, service, price, etc.)<br>2. OpenAI generates a personalized intro paragraph and scope summary<br>3. Copies the Google Docs template to a new document<br>4. Replaces all {{variables}} in the copied doc with form data + AI content<br>5. Exports the document as PDF<br>6. Sends the PDF as an email attachment via Gmail<br>### Setup Required<br>1. Create a Google Docs template with these variables:<br>{{client_name}}, {{company_name}}, {{client_email}}, {{service}}, {{price}}, {{valid_until}}, {{notes}}, {{intro}}, {{scope_summary}}<br>2. Copy the template Document ID and paste it in the Copy Template node<br>3. Configure Google Docs OAuth credential in Replace Variables node<br>4. Configure Gmail OAuth credential in Send Quotation Email node<br>### Credentials Required<br>- OpenAI API key<br>- Google Drive OAuth<br>- Google Docs OAuth<br>- Gmail OAuth |
| Export as PDF | n8n-nodes-base.googleDrive | Downloads the completed Google Doc as a PDF binary file | Replace Variables | Send Quotation Email | ## Quotation Generator PDF<br>Generates personalized quotations from a form submission, creates a branded PDF, and emails it to the client.<br>### Flow<br>Form > OpenAI Personalize > Copy Template > Replace Variables > Export PDF > Send Email<br>### How It Works<br>1. Client fills out the quotation form (name, email, company, service, price, etc.)<br>2. OpenAI generates a personalized intro paragraph and scope summary<br>3. Copies the Google Docs template to a new document<br>4. Replaces all {{variables}} in the copied doc with form data + AI content<br>5. Exports the document as PDF<br>6. Sends the PDF as an email attachment via Gmail<br>### Setup Required<br>1. Create a Google Docs template with these variables:<br>{{client_name}}, {{company_name}}, {{client_email}}, {{service}}, {{price}}, {{valid_until}}, {{notes}}, {{intro}}, {{scope_summary}}<br>2. Copy the template Document ID and paste it in the Copy Template node<br>3. Configure Google Docs OAuth credential in Replace Variables node<br>4. Configure Gmail OAuth credential in Send Quotation Email node<br>### Credentials Required<br>- OpenAI API key<br>- Google Drive OAuth<br>- Google Docs OAuth<br>- Gmail OAuth |
| Send Quotation Email | n8n-nodes-base.gmail | Sends the quotation email with PDF attachment to the client | Export as PDF |  | ## Quotation Generator PDF<br>Generates personalized quotations from a form submission, creates a branded PDF, and emails it to the client.<br>### Flow<br>Form > OpenAI Personalize > Copy Template > Replace Variables > Export PDF > Send Email<br>### How It Works<br>1. Client fills out the quotation form (name, email, company, service, price, etc.)<br>2. OpenAI generates a personalized intro paragraph and scope summary<br>3. Copies the Google Docs template to a new document<br>4. Replaces all {{variables}} in the copied doc with form data + AI content<br>5. Exports the document as PDF<br>6. Sends the PDF as an email attachment via Gmail<br>### Setup Required<br>1. Create a Google Docs template with these variables:<br>{{client_name}}, {{company_name}}, {{client_email}}, {{service}}, {{price}}, {{valid_until}}, {{notes}}, {{intro}}, {{scope_summary}}<br>2. Copy the template Document ID and paste it in the Copy Template node<br>3. Configure Google Docs OAuth credential in Replace Variables node<br>4. Configure Gmail OAuth credential in Send Quotation Email node<br>### Credentials Required<br>- OpenAI API key<br>- Google Drive OAuth<br>- Google Docs OAuth<br>- Gmail OAuth |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation and setup guidance |  |  | ## Quotation Generator PDF<br>Generates personalized quotations from a form submission, creates a branded PDF, and emails it to the client.<br>### Flow<br>Form > OpenAI Personalize > Copy Template > Replace Variables > Export PDF > Send Email<br>### How It Works<br>1. Client fills out the quotation form (name, email, company, service, price, etc.)<br>2. OpenAI generates a personalized intro paragraph and scope summary<br>3. Copies the Google Docs template to a new document<br>4. Replaces all {{variables}} in the copied doc with form data + AI content<br>5. Exports the document as PDF<br>6. Sends the PDF as an email attachment via Gmail<br>### Setup Required<br>1. Create a Google Docs template with these variables:<br>{{client_name}}, {{company_name}}, {{client_email}}, {{service}}, {{price}}, {{valid_until}}, {{notes}}, {{intro}}, {{scope_summary}}<br>2. Copy the template Document ID and paste it in the Copy Template node<br>3. Configure Google Docs OAuth credential in Replace Variables node<br>4. Configure Gmail OAuth credential in Send Quotation Email node<br>### Credentials Required<br>- OpenAI API key<br>- Google Drive OAuth<br>- Google Docs OAuth<br>- Gmail OAuth |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `Quotation Generator PDF`.
   - Keep the workflow inactive until credentials and template IDs are fully configured.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Set the form title to `Quotation Request`.
   - Set the form description to `Fill in the details below to generate a personalized quotation.`
   - Add these fields:
     1. `Client Name` — string — required — placeholder `John Smith`
     2. `Client Email` — string — required — placeholder `user@example.com`
     3. `Company Name` — string — required — placeholder `Acme Corp`
     4. `Service` — string — required — placeholder `e.g. CRM Automation, Lead Pipeline, AI Chatbot`
     5. `Service Description` — text/string field — required — placeholder `Describe the scope of work and deliverables`
     6. `Price` — string — required — placeholder `e.g. $2,500`
     7. `Valid Until` — string — required — placeholder `e.g. April 30, 2026`
     8. `Notes` — optional — placeholder `Any additional terms or notes`
   - Set **Response Mode** to `Last Node`.

3. **Add an OpenAI node**
   - Node type: **OpenAI** from the LangChain/OpenAI integration
   - Name it `OpenAI Personalize`.
   - Connect **Quotation Form → OpenAI Personalize**.
   - Configure credentials:
     - Add an **OpenAI API** credential with a valid API key.
   - Set model to `gpt-4o-mini`.
   - Set temperature to `0.7`.
   - Add a **system message**:
     - `You are a professional business proposal writer for Baraco AI, an automation agency specializing in n8n workflows and AI integration.`
   - Add a **user message** instructing the model to create:
     - a personalized intro
     - a scope summary
     - strict JSON output
   - Use the submitted form fields in expressions:
     - Client Name
     - Company Name
     - Service
     - Service Description
     - Notes
   - Enable **JSON output**.

4. **Prepare the Google Docs quotation template**
   - In Google Docs, create a master quotation document.
   - Include these placeholders exactly:
     - `{{client_name}}`
     - `{{company_name}}`
     - `{{client_email}}`
     - `{{service}}`
     - `{{price}}`
     - `{{valid_until}}`
     - `{{notes}}`
     - `{{intro}}`
     - `{{scope_summary}}`
   - Save the file in Google Drive.
   - Copy the document ID from the URL.

5. **Add a Google Drive node to duplicate the template**
   - Node type: **Google Drive**
   - Name it `Copy Template`.
   - Connect **OpenAI Personalize → Copy Template**.
   - Configure Google Drive OAuth2 credentials with access to the template file.
   - Set operation to `Copy`.
   - Set the source file ID to the Google Docs template ID from step 4.
   - Set the new file name dynamically:
     - `Quotation - {{ Company Name }} - {{ Service }}`
   - Use expressions referencing the Form Trigger node for company and service.

6. **Add a Google Docs node to replace placeholders**
   - Node type: **Google Docs**
   - Name it `Replace Variables`.
   - Connect **Copy Template → Replace Variables**.
   - Configure Google Docs OAuth2 credentials.
   - Set operation to `Update`.
   - Add one replace action for each placeholder:
     - `{{client_name}}` → Client Name
     - `{{company_name}}` → Company Name
     - `{{client_email}}` → Client Email
     - `{{service}}` → Service
     - `{{price}}` → Price
     - `{{valid_until}}` → Valid Until
     - `{{notes}}` → Notes
     - `{{intro}}` → OpenAI intro
     - `{{scope_summary}}` → OpenAI scope summary
   - Ensure the node targets the copied document produced by `Copy Template`.
   - If your n8n version requires an explicit document ID field, map the ID from the previous Google Drive copy node.

7. **Add a Google Drive node to export the document as PDF**
   - Node type: **Google Drive**
   - Name it `Export as PDF`.
   - Connect **Replace Variables → Export as PDF**.
   - Use the same Google Drive OAuth2 credential as the copy step if appropriate.
   - Set operation to `Download`.
   - Set the file ID to the ID of the copied Google Doc from `Copy Template`.
   - Enable Google file conversion.
   - Choose conversion format:
     - Google Docs → `application/pdf`

8. **Add a Gmail node to send the quotation**
   - Node type: **Gmail**
   - Name it `Send Quotation Email`.
   - Connect **Export as PDF → Send Quotation Email**.
   - Configure Gmail OAuth2 credentials with permission to send mail.
   - Set recipient to the submitted client email.
   - Set subject dynamically to:
     - `Quotation from Baraco AI - {{ Service }}`
   - Compose the email body with:
     - greeting using client name
     - AI introduction
     - statement that the quotation is attached
     - scope of work
     - price
     - validity date
     - Baraco AI signature
   - Enable binary attachment sending.
   - Map the PDF binary output from the previous node as the attachment.

9. **Add a Sticky Note for operational context**
   - Node type: **Sticky Note**
   - Paste a summary describing:
     - what the workflow does
     - the node flow
     - required placeholders
     - required credentials

10. **Validate all expressions**
   - Confirm the OpenAI node returns JSON in the structure expected by later nodes.
   - Check whether your OpenAI node stores results under `message.content`, `content`, or another path in your installed version.
   - Update downstream expressions if the structure differs.

11. **Test end-to-end**
   - Submit the form with sample values.
   - Verify:
     - OpenAI returns valid JSON
     - a new Google Doc copy is created
     - placeholders are replaced
     - PDF is generated
     - email is sent with the PDF attached

12. **Activate the workflow**
   - Only activate after confirming:
     - template ID is valid
     - all OAuth credentials are connected
     - email attachment is included correctly
     - optional blank notes do not break formatting

## Credential Configuration Summary

### OpenAI
- Required for `OpenAI Personalize`
- Must allow chat/completion access for the selected model

### Google Drive OAuth2
- Required for:
  - `Copy Template`
  - `Export as PDF`
- Must allow file copy and file download/export

### Google Docs OAuth2
- Required for:
  - `Replace Variables`
- Must allow editing Google Docs content

### Gmail OAuth2
- Required for:
  - `Send Quotation Email`
- Must allow sending mail from the connected account

## Expected Input/Output Behavior

### Workflow input
One form submission containing:
- Client Name
- Client Email
- Company Name
- Service
- Service Description
- Price
- Valid Until
- Notes

### Intermediate outputs
- OpenAI node returns structured JSON with:
  - `intro`
  - `scope_summary`
- Google Drive copy node returns the new Google Doc file ID
- Google Drive export node returns PDF binary data

### Final output
- Sent email with:
  - personalized body text
  - attached quotation PDF

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow uses Google Docs as a template engine rather than generating PDFs directly from HTML or code. | General architecture |
| The Google Docs template must contain the exact placeholder names used in the replacement step. | Template preparation |
| The value `REPLACE_WITH_TEMPLATE_DOC_ID` is a manual setup placeholder and must be replaced before execution. | Copy Template node |
| The workflow is configured with `binaryMode: separate`, which is relevant for attachment handling in the Gmail step. | Workflow setting |
| The Gmail and Google Docs credentials are labeled in a way that may not match their actual service names; verify the underlying credential type, not only the label. | Credential hygiene |
| Because the Form Trigger responds from the last node, any failure in document generation or email sending may be exposed back to the form submitter as a failed submission. | Runtime behavior |
| No external blog links, branding links, or video links were included in the provided workflow data. | Source workflow content |