Automated Digital Certificate Creator & Validator with PDF Generation

https://n8nworkflows.xyz/workflows/automated-digital-certificate-creator---validator-with-pdf-generation-11097


# Automated Digital Certificate Creator & Validator with PDF Generation

### 1. Workflow Overview

This workflow automates the creation and validation of digital training certificates with PDF generation and email delivery. It serves two main use cases:

- **Certificate Creation:** Accepts learner details (name, surname, course, email) via an HTTP POST request, generates a unique certification ID, verifies its uniqueness, stores the certificate record, generates a styled PDF certificate, and emails it to the learner.
- **Certificate Validation:** Accepts a certification ID via HTTP POST, looks it up in the stored records, and returns JSON indicating if the certificate exists along with associated learner details.

The workflow is logically divided into two primary functional blocks:

- **1.1 Certificate Creation Block:** Handles input reception, unique ID generation, duplicate check, data storage, PDF generation, and email sending.
- **1.2 Certificate Validation Block:** Handles lookup of a certification ID and returns a verification response.

Additional logical groupings include webhook input reception nodes and response nodes to handle HTTP communication.

---

### 2. Block-by-Block Analysis

#### 1.1 Certificate Creation Block

**Overview:**  
This block manages the end-to-end process of generating a unique certification ID, validating it against existing records to avoid duplicates, storing the certificate data, generating a PDF certificate, and emailing it to the learner.

**Nodes Involved:**
- Webhook_Creation
- Generate_Certification_ID
- Find_Certification_By_ID
- Certification_ID_Exists (IF node)
- Insert_Certificaton
- Generate_PDF
- Email_Certification

**Node Details:**

- **Webhook_Creation**  
  - *Type:* Webhook  
  - *Role:* Entry point that receives HTTP POST requests at path `/certifications`. Collects learner data in headers: name, surname, course, email.  
  - *Config:* HTTP Method POST, responseMode “lastNode” (responds after the last node finishes).  
  - *Connections:* Outputs to Generate_Certification_ID.  
  - *Failure Points:* Misconfigured webhook path, missing headers, malformed requests.  
  - *Notes:* Starts the certificate creation flow.

- **Generate_Certification_ID**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Generates a unique CertificationID by combining a timestamp-based string and a random alphanumeric segment, both uppercased.  
  - *Config:* Returns an object `{id: uniqueId}` with the generated ID.  
  - *Connections:* Outputs to Find_Certification_By_ID.  
  - *Failure Points:* Very low risk, but theoretically could generate duplicate IDs due to randomness.  
  - *Notes:* Key to uniqueness of certificates.

- **Find_Certification_By_ID**  
  - *Type:* Data Table Node  
  - *Role:* Queries the data table “Certifications” to check if the generated CertificationID already exists.  
  - *Config:* Filter condition on `CertificationID` equals the generated ID. Returns all matching records.  
  - *Connections:* Outputs to Certification_ID_Exists IF node.  
  - *Failure Points:* Data Table not found, connectivity issues, query errors.

- **Certification_ID_Exists**  
  - *Type:* IF Node  
  - *Role:* Checks if the query result from the previous node is empty (no existing CertificationID) or not.  
  - *Config:* Condition checks if the output JSON is empty (no match).  
  - *Connections:*  
    - If ID exists (non-empty): loops back to Generate_Certification_ID node to generate another ID.  
    - If ID does not exist (empty): proceeds to Insert_Certificaton node.  
  - *Failure Points:* Expression evaluation failure; infinite loop risk if very unlucky with ID generation.

- **Insert_Certificaton**  
  - *Type:* Data Table Node  
  - *Role:* Inserts a new record into the “Certifications” Data Table with learner name, surname, and CertificationID.  
  - *Config:* Columns are populated from webhook headers and generated CertificationID.  
  - *Connections:* Outputs to Generate_PDF node.  
  - *Failure Points:* Data Table write errors, missing data from previous nodes.

- **Generate_PDF**  
  - *Type:* PDF Generator API Node  
  - *Role:* Creates a PDF certificate from an HTML template styled with CSS and populated with learner details and CertificationID.  
  - *Config:*  
    - HTML content includes placeholders for name, surname, course, CertificationID, and current date formatted as dd/MM/yyyy.  
    - Output is a file attachment.  
    - Page orientation: landscape.  
    - Uses PDF Generator API credentials.  
  - *Connections:* Outputs PDF binary to Email_Certification node.  
  - *Failure Points:* API authentication errors, template rendering issues, malformed HTML, API rate limits.

- **Email_Certification**  
  - *Type:* Gmail Node (OAuth2)  
  - *Role:* Sends the generated PDF certificate as an email attachment to the learner’s email address.  
  - *Config:*  
    - Recipient address from webhook header `email`.  
    - Subject: “Your certification is ready!”  
    - Message body: “Attached you find your new certification!”  
    - Attachment: PDF from Generate_PDF node (binary property named `document.pdf`).  
    - Uses Gmail OAuth2 credentials.  
  - *Failure Points:* Email sending errors, invalid email addresses, OAuth token expiry, attachment size limits.

---

#### 1.2 Certificate Validation Block

**Overview:**  
This block allows external clients to verify the validity of a certificate by submitting a CertificationID. It queries the data table and returns a JSON response indicating whether the certificate exists along with the learner’s name and surname.

**Nodes Involved:**
- Webhook_Check
- Find_Certification_By_ID1
- Certification_Exists (IF node)
- Respond_Found
- Respond_NotFound

**Node Details:**

- **Webhook_Check**  
  - *Type:* Webhook  
  - *Role:* Entry point that receives HTTP POST requests at path `/certificationscheck` with CertificationID in headers.  
  - *Config:* HTTP Method POST, responseMode “responseNode” (responds explicitly from Respond nodes).  
  - *Connections:* Outputs to Find_Certification_By_ID1.  
  - *Failure Points:* Misconfigured webhook path, missing or malformed CertificationID.

- **Find_Certification_By_ID1**  
  - *Type:* Data Table Node  
  - *Role:* Queries “Certifications” Data Table for the CertificationID sent by client.  
  - *Config:* Filter on `CertificationID` equals header `id`. Returns all matches.  
  - *Connections:* Outputs to Certification_Exists IF node.  
  - *Failure Points:* Data Table access errors.

- **Certification_Exists**  
  - *Type:* IF Node  
  - *Role:* Checks if the CertificationID from the query matches the requested ID to determine existence.  
  - *Config:* Condition tests equality between returned CertificationID and requested ID.  
  - *Connections:*  
    - If true: proceeds to Respond_Found node.  
    - If false: proceeds to Respond_NotFound node.  
  - *Failure Points:* Expression errors, unexpected data shape.

- **Respond_Found**  
  - *Type:* Respond to Webhook Node  
  - *Role:* Returns a JSON response with `ok:true` and learner’s name and surname for valid CertificationIDs.  
  - *Config:* Responds with JSON body including name and surname extracted from the Data Table node.  
  - *Failure Points:* Webhook response errors.

- **Respond_NotFound**  
  - *Type:* Respond to Webhook Node  
  - *Role:* Returns a JSON response with `ok:false` indicating the CertificationID was not found.  
  - *Config:* Static JSON response.  
  - *Failure Points:* Webhook response errors.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                          | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                  |
|---------------------------|--------------------------------|----------------------------------------|---------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Webhook_Creation          | Webhook                        | Receives creation input via HTTP POST  | —                         | Generate_Certification_ID   | ## Creation  \n## Webhook input\n\nReceives name, surname, course and email via HTTP POST on `/certifications`.  \nStarts the certificate creation flow. |
| Generate_Certification_ID  | Code                          | Generates unique CertificationID       | Webhook_Creation          | Find_Certification_By_ID    | ## Creation – Generate & validate ID\n\nGenerates a random CertificationID, checks the Data Table for duplicates and only continues when the ID is unique.  \nIf the ID already exists, the flow loops until a free one is found. |
| Find_Certification_By_ID   | Data Table                    | Checks if CertificationID exists       | Generate_Certification_ID | Certification_ID_Exists     |                                                                                              |
| Certification_ID_Exists    | IF                            | Branches based on CertificationID uniqueness | Find_Certification_By_ID  | Generate_Certification_ID (if exists), Insert_Certificaton (if unique) |                                                                                              |
| Insert_Certificaton        | Data Table                    | Inserts new certificate record          | Certification_ID_Exists   | Generate_PDF                | ## Creation – Save, PDF & email\n\nSaves the learner record with the CertificationID to the Data Table, generates the certificate PDF from an HTML template and emails it to the learner. |
| Generate_PDF               | PDF Generator API             | Generates PDF certificate from HTML    | Insert_Certificaton       | Email_Certification         |                                                                                              |
| Email_Certification        | Gmail (OAuth2)                | Emails the generated PDF to learner    | Generate_PDF              | —                          |                                                                                              |
| Webhook_Check             | Webhook                        | Receives validation input via HTTP POST | —                         | Find_Certification_By_ID1   | ## Check – Webhook input\n\nReceives a CertificationID via HTTP POST on `/certificationscheck` and starts the verification flow. |
| Find_Certification_By_ID1  | Data Table                    | Finds certificate by CertificationID  | Webhook_Check             | Certification_Exists        | ## Check – Look up ID\n\nSearches the Certifications Data Table for the submitted CertificationID and branches depending on whether a matching record exists. |
| Certification_Exists       | IF                            | Branches based on certificate existence | Find_Certification_By_ID1 | Respond_Found (if exists), Respond_NotFound (if not) |                                                                                              |
| Respond_Found              | Respond to Webhook            | Returns valid certificate info JSON    | Certification_Exists      | —                          | ## Check – Responses\n\nReturns JSON to the caller.  \nIf the ID is found: `ok: true` plus name and surname.  \nIf not found: `ok: false` so the client can show an “invalid certification” message. |
| Respond_NotFound           | Respond to Webhook            | Returns invalid certificate JSON       | Certification_Exists      | —                          |                                                                                              |
| Sticky Note1               | Sticky Note                   | Comment block for Creation webhook     | —                         | —                          | ## Creation  \n## Webhook input\n\nReceives name, surname, course and email via HTTP POST on `/certifications`.  \nStarts the certificate creation flow. |
| Sticky Note2               | Sticky Note                   | Comment block for ID generation         | —                         | —                          | ## Creation – Generate & validate ID\n\nGenerates a random CertificationID, checks the Data Table for duplicates and only continues when the ID is unique.  \nIf the ID already exists, the flow loops until a free one is found. |
| Sticky Note4               | Sticky Note                   | Comment block for save, PDF & email     | —                         | —                          | ## Creation – Save, PDF & email\n\nSaves the learner record with the CertificationID to the Data Table, generates the certificate PDF from an HTML template and emails it to the learner. |
| Sticky Note6               | Sticky Note                   | Comment block for Check webhook         | —                         | —                          | ## Check – Webhook input\n\nReceives a CertificationID via HTTP POST on `/certificationscheck` and starts the verification flow. |
| Sticky Note7               | Sticky Note                   | Comment block for lookup logic          | —                         | —                          | ## Check – Look up ID\n\nSearches the Certifications Data Table for the submitted CertificationID and branches depending on whether a matching record exists. |
| Sticky Note8               | Sticky Note                   | Comment block for response logic        | —                         | —                          | ## Check – Responses\n\nReturns JSON to the caller.  \nIf the ID is found: `ok: true` plus name and surname.  \nIf not found: `ok: false` so the client can show an “invalid certification” message. |
| Sticky Note10              | Sticky Note                   | Overall workflow explanation and setup | —                         | —                          | ## Certifications – Creator & Checker\n\n## How it works\nThis workflow manages the full lifecycle of a training certificate.  \n`/certifications` receives learner details, generates a unique CertificationID and checks that it does not exist yet.  \nIf the ID is free, the workflow stores the record, builds a PDF certificate and emails it to the learner.  \n\n`/certificationscheck` lets you verify a certificate later.  \nThe webhook receives a CertificationID, looks it up in the same Data Table and returns a simple JSON response that frontends or other services can use to show “valid” or “not found”.\n\n## Setup steps\n1. Create a Data Table (e.g. **Certifications**) with Name, Surname, CertificationID and any extra fields.\n2. Configure the PDF Generator node with your account API and adjust the HTML template to your branding.\n3. Connect your email account (Gmail/SMTP) in **Email_Certification**.\n4. Deploy the `/certifications` and `/certificationscheck` webhooks and use their public URLs.\n5. Test creation and verification with a few sample requests before going live. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Data Table:**  
   - Name: `Certifications`  
   - Columns: `Name` (string), `Surname` (string), `CertificationID` (string)  
   - Include any additional fields as needed.

2. **Create Webhook Node for Creation Input:**  
   - Node Name: `Webhook_Creation`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `certifications`  
   - Response Mode: lastNode  
   - Expect headers: `name`, `surname`, `course`, `email`

3. **Create Code Node to Generate Unique CertificationID:**  
   - Node Name: `Generate_Certification_ID`  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const uniqueId =
       Date.now().toString(36).toUpperCase() +
       Math.random().toString(36).substring(2, 8).toUpperCase();
     return [{ id: uniqueId }];
     ```
   - Connect output of `Webhook_Creation` to this node.

4. **Create Data Table Node to Check if CertificationID Exists:**  
   - Node Name: `Find_Certification_By_ID`  
   - Type: Data Table  
   - Operation: Get  
   - Data Table ID: your `Certifications` table ID  
   - Filter: `CertificationID` equals `={{ $json.id }}` (output from Code node)  
   - Connect output of `Generate_Certification_ID` to this node.

5. **Create IF Node to Check if ID is Unique:**  
   - Node Name: `Certification_ID_Exists`  
   - Type: IF  
   - Condition: Check if the output from `Find_Certification_By_ID` is empty (no records found)  
   - If ID exists (not empty), connect back to `Generate_Certification_ID` to retry.  
   - If ID is unique (empty), connect to `Insert_Certificaton`.

6. **Create Data Table Node to Insert New Certification Record:**  
   - Node Name: `Insert_Certificaton`  
   - Type: Data Table  
   - Operation: Insert  
   - Data Table ID: `Certifications`  
   - Columns:  
     - `Name`: `={{ $('Webhook_Creation').item.json.headers.name }}`  
     - `Surname`: `={{ $('Webhook_Creation').item.json.headers.surname }}`  
     - `CertificationID`: `={{ $('Generate_Certification_ID').item.json.id }}`  
   - Connect output of IF node’s "unique" branch to this node.

7. **Create PDF Generator Node:**  
   - Node Name: `Generate_PDF`  
   - Type: PDF Generator API  
   - Resource: Conversion  
   - HTML Content: Use the given styled certificate HTML template with dynamic placeholders for name, surname, course, CertificationID, and date.  
   - Conversion Options: Landscape orientation, output file format.  
   - Credentials: Configure with your PDF Generator API account.  
   - Connect output of `Insert_Certificaton` to this node.

8. **Create Email Node to Send Certificate:**  
   - Node Name: `Email_Certification`  
   - Type: Gmail (OAuth2) or SMTP  
   - To: `={{ $('Webhook_Creation').item.json.headers.email }}`  
   - Subject: “Your certification is ready!”  
   - Message: “Attached you find your new certification!”  
   - Attachment: PDF binary output from `Generate_PDF` (property `document.pdf`)  
   - Credentials: Set up Gmail OAuth2 or SMTP credentials.  
   - Connect output of `Generate_PDF` to this node.

---

**Validation Flow:**

9. **Create Webhook Node for Validation Input:**  
   - Node Name: `Webhook_Check`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `certificationscheck`  
   - Response Mode: responseNode  
   - Expect header: `id` (CertificationID)

10. **Create Data Table Node to Find Certification by ID:**  
    - Node Name: `Find_Certification_By_ID1`  
    - Type: Data Table  
    - Operation: Get  
    - Data Table ID: `Certifications`  
    - Filter: `CertificationID` equals `={{ $json.headers.id }}`  
    - Connect output of `Webhook_Check` to this node.

11. **Create IF Node to Check if Certification Exists:**  
    - Node Name: `Certification_Exists`  
    - Type: IF  
    - Condition: Check if the returned `CertificationID` equals the requested ID from webhook header.  
    - If true, connect to Respond_Found node.  
    - If false, connect to Respond_NotFound node.

12. **Create Respond to Webhook Node for Found Certificates:**  
    - Node Name: `Respond_Found`  
    - Type: Respond to Webhook  
    - Respond With: JSON  
    - Response Body:  
      ```json
      {
        "ok": "true",
        "name": "{{ $('Find_Certification_By_ID1').item.json.Name }}",
        "surname": "{{ $('Find_Certification_By_ID1').item.json.Surname }}"
      }
      ```
    - Connect "true" branch of IF node here.

13. **Create Respond to Webhook Node for Not Found Certificates:**  
    - Node Name: `Respond_NotFound`  
    - Type: Respond to Webhook  
    - Respond With: JSON  
    - Response Body: `{"ok": "false"}`  
    - Connect "false" branch of IF node here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow manages the full lifecycle of a training certificate: generation, storage, PDF creation, email delivery, and later verification via API.                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note10 content in the workflow                                                                       |
| Setup steps include creating a Data Table named `Certifications` with required fields, configuring PDF Generator API credentials, connecting a Gmail or SMTP account for email sending, and deploying webhooks at `/certifications` and `/certificationscheck`. Testing before production use is recommended.                                                                                                                                                                                                                                 | Sticky Note10 content in the workflow                                                                       |
| The PDF certificate HTML template is styled with CSS for a professional look, using placeholders for dynamic content such as name, surname, course, CertificationID, and current date. It uses a landscape orientation and a gradient border for branding.                                                                                                                                                                                                                                                                            | HTML template inside Generate_PDF node                                                                       |
| Email sending uses Gmail OAuth2 credentials; ensure OAuth tokens are valid and the account allows sending emails programmatically. Attachment size limits and email spam filtering should be considered.                                                                                                                                                                                                                                                                                                                               | Email_Certification node configuration                                                                       |
| The ID generation uses a combination of timestamp and random alphanumeric characters, but theoretically collisions could occur. The workflow loops until a unique ID is found to prevent duplicates, but extensive collisions could cause delays.                                                                                                                                                                                                                                                                                     | Sticky Note2 and Generate_Certification_ID node logic                                                       |
| Ensure all webhook URLs are publicly accessible and secured appropriately (e.g., via API gateway or firewall) to avoid unauthorized certificate creation or validation requests.                                                                                                                                                                                                                                                                                                                                                        | General security best practice                                                                                |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.