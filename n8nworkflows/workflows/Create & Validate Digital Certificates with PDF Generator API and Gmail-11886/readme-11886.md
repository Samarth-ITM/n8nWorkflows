Create & Validate Digital Certificates with PDF Generator API and Gmail

https://n8nworkflows.xyz/workflows/create---validate-digital-certificates-with-pdf-generator-api-and-gmail-11886


# Create & Validate Digital Certificates with PDF Generator API and Gmail

### 1. Workflow Overview

This workflow automates the creation, validation, and distribution of digital training certificates using n8n. It supports two main use cases:  
- **Certificate Creation:** Receives learner details via a webhook, generates a unique CertificationID, validates its uniqueness, stores the certificate data, generates a PDF certificate from a template, and emails it to the recipient.  
- **Certificate Verification:** Receives a CertificationID via a separate webhook, checks its existence in the data storage, and returns a JSON response indicating whether the certificate is valid along with the associated learner’s name and surname.

The workflow is logically divided into two main blocks:

- **1.1 Certificate Creation Block:** Handles input reception, unique ID generation and validation, data storage, PDF generation, and email dispatch.  
- **1.2 Certificate Verification Block:** Handles input reception for verification, look-up of the CertificationID, and returns a structured JSON response indicating validity.

The workflow uses n8n Data Tables to store certificate records, a PDF Generator API node to create visually branded certificates, and Gmail OAuth2 credentials for email sending.

---

### 2. Block-by-Block Analysis

#### 1.1 Certificate Creation Block

**Overview:**  
Processes incoming POST requests with learner details, generates a unique CertificationID, validates uniqueness, saves the data, generates the PDF certificate, and sends it via email.

**Nodes Involved:**  
- Webhook_Creation  
- Generate_Certification_ID  
- Find_Certification_By_ID  
- Certification_ID_Exists (If node)  
- Insert_Certificaton (Data Table)  
- Generate a PDF document (PDF Generator API)  
- Email_Certification (Gmail)  
- Sticky Note1, Sticky Note2, Sticky Note4 (documentation aids)

**Node Details:**

- **Webhook_Creation**  
  - Type: Webhook  
  - Role: Entry point for creation flow; receives HTTP POST on `/certifications2`.  
  - Config: POST method, path `certifications2`, response mode is last node (the email node).  
  - Inputs: HTTP request with headers containing `name`, `surname`, `email`, `course`.  
  - Outputs: Trigger to `Generate_Certification_ID`.  
  - Failures: HTTP errors, missing headers, malformed requests.

- **Generate_Certification_ID**  
  - Type: Code (JavaScript)  
  - Role: Generates a random unique CertificationID combining timestamp and random string.  
  - Config: Generates a 12-character uppercase string (timestamp + random substring).  
  - Inputs: Trigger from webhook.  
  - Outputs: Passes ID to `Find_Certification_By_ID`.  
  - Edge cases: Possible collision (very low probability), handled by re-checking in next node.

- **Find_Certification_By_ID**  
  - Type: Data Table (get operation)  
  - Role: Checks if generated CertificationID already exists in `Certifications` Data Table.  
  - Config: Filter by `CertificationID == generated id`, returns all matches.  
  - Inputs: CertificationID from previous node.  
  - Outputs: To `Certification_ID_Exists`.  
  - Errors: Data Table connection issues, empty responses.

- **Certification_ID_Exists** (If node)  
  - Type: If  
  - Role: Evaluates if the CertificationID exists by checking if the Data Table response is empty.  
  - Config: Condition checks if output JSON is empty (true if no existing record).  
  - Inputs: Output of `Find_Certification_By_ID`.  
  - Outputs:  
    - True (ID does not exist): continues to `Insert_Certificaton`.  
    - False (ID exists): loops back to `Generate_Certification_ID` to generate a new ID.  
  - Edge cases: Infinite loop protection recommended if many collisions occur.

- **Insert_Certificaton**  
  - Type: Data Table (insert operation)  
  - Role: Saves learner’s name, surname, and CertificationID into the `Certifications` Data Table.  
  - Config: Inserts columns `Name`, `Surname`, `CertificationID` using values from webhook and generated ID.  
  - Inputs: Data from webhook and ID generator.  
  - Outputs: To `Generate a PDF document`.  
  - Failures: Data Table write permissions, connectivity.

- **Generate a PDF document**  
  - Type: PDF Generator API node  
  - Role: Generates a PDF certificate based on an HTML template and dynamic data.  
  - Config: Uses template ID `1560735`, fills placeholders with current date, candidate name, course, and CertificationID.  
  - Inputs: Data from `Insert_Certificaton` and webhook.  
  - Outputs: PDF file in binary property `document.pdf` to `Email_Certification`.  
  - Failures: API authentication errors, template misconfiguration, network issues.

- **Email_Certification**  
  - Type: Gmail node  
  - Role: Sends the generated PDF certificate via email to the learner.  
  - Config: Uses Gmail OAuth2 credentials, sends to email provided in webhook headers, subject “Your certification is ready!”, attaches generated PDF.  
  - Inputs: PDF binary data and email address.  
  - Outputs: HTTP response for webhook completion.  
  - Failures: OAuth token expiration, email sending limits, invalid email address.

- **Sticky Notes in this block:** Provide contextual documentation about input reception, ID generation and validation, and saving/PDF/email steps.

---

#### 1.2 Certificate Verification Block

**Overview:**  
Receives CertificationID for verification via webhook, searches the Data Table, and responds with JSON indicating validity and learner details if found.

**Nodes Involved:**  
- Webhook_Check  
- Find_Certification_By_ID1  
- Certification_Exists (If node)  
- Respond_Found  
- Respond_NotFound  
- Sticky Note6, Sticky Note7, Sticky Note8 (documentation aids)

**Node Details:**

- **Webhook_Check**  
  - Type: Webhook  
  - Role: Entry point for verification flow; receives HTTP POST on `/certificationscheck2`.  
  - Config: POST method, path `certificationscheck2`, responseMode `responseNode` for direct response.  
  - Inputs: HTTP request headers containing `id` (CertificationID).  
  - Outputs: To `Find_Certification_By_ID1`.  
  - Failures: HTTP errors, missing or malformed ID.

- **Find_Certification_By_ID1**  
  - Type: Data Table (get operation)  
  - Role: Looks up the `Certifications` Data Table for the provided CertificationID.  
  - Config: Filter by `CertificationID == received id`, returns all matches.  
  - Inputs: CertificationID from webhook.  
  - Outputs: To `Certification_Exists`.  
  - Failures: Data Table unavailability.

- **Certification_Exists** (If node)  
  - Type: If  
  - Role: Checks if the CertificationID exists by comparing the CertificationID from Data Table to the header id.  
  - Config: Condition checks equality of IDs.  
  - Inputs: Data Table lookup result.  
  - Outputs:  
    - True: To `Respond_Found`.  
    - False: To `Respond_NotFound`.

- **Respond_Found**  
  - Type: Respond to Webhook  
  - Role: Returns JSON response indicating success and includes learner’s name and surname.  
  - Config: Responds with JSON: `{ "ok": "true", "name": "...", "surname": "..." }` using data from Data Table.  
  - Inputs: Positive existence from If node.  
  - Outputs: HTTP response to client.  
  - Failures: Response formatting errors.

- **Respond_NotFound**  
  - Type: Respond to Webhook  
  - Role: Returns JSON response indicating failure (certificate not found).  
  - Config: Responds with JSON: `{ "ok": "false" }`.  
  - Inputs: Negative existence from If node.  
  - Outputs: HTTP response to client.  
  - Failures: None expected.

- **Sticky Notes in this block:** Explain webhook input, lookup logic, and response format.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                                    | Input Node(s)          | Output Node(s)                  | Sticky Note                                                  |
|-------------------------|----------------------------|---------------------------------------------------|------------------------|--------------------------------|--------------------------------------------------------------|
| Webhook_Creation        | Webhook                    | Receives creation requests                         | —                      | Generate_Certification_ID       | ## Creation – Webhook input                                  |
| Generate_Certification_ID | Code                      | Generates unique CertificationID                  | Webhook_Creation       | Find_Certification_By_ID        | ## Creation – Generate & validate ID                        |
| Find_Certification_By_ID | Data Table                 | Checks if CertificationID exists                   | Generate_Certification_ID | Certification_ID_Exists        | ## Creation – Generate & validate ID                        |
| Certification_ID_Exists  | If                         | Determines if CertificationID is unique           | Find_Certification_By_ID | Generate_Certification_ID (if exists), Insert_Certificaton (if new) | ## Creation – Generate & validate ID                        |
| Insert_Certificaton      | Data Table                 | Saves new certificate record                        | Certification_ID_Exists | Generate a PDF document         | ## Creation – Save, PDF & email                             |
| Generate a PDF document  | PDF Generator API          | Creates PDF certificate document                    | Insert_Certificaton    | Email_Certification             | ## Creation – Save, PDF & email                             |
| Email_Certification      | Gmail                      | Sends the PDF certificate by email                  | Generate a PDF document | Webhook_Creation (response)     | ## Creation – Save, PDF & email                             |
| Webhook_Check            | Webhook                    | Receives verification requests                      | —                      | Find_Certification_By_ID1       | ## Check – Webhook input                                    |
| Find_Certification_By_ID1 | Data Table                | Looks up CertificationID for verification           | Webhook_Check          | Certification_Exists            | ## Check – Look up ID                                       |
| Certification_Exists     | If                         | Checks if CertificationID exists                    | Find_Certification_By_ID1 | Respond_Found (if true), Respond_NotFound (if false) | ## Check – Look up ID                                       |
| Respond_Found            | Respond to Webhook         | Responds with success and learner details           | Certification_Exists   | —                              | ## Check – Responses                                       |
| Respond_NotFound         | Respond to Webhook         | Responds with failure (certificate not found)       | Certification_Exists   | —                              | ## Check – Responses                                       |
| Sticky Note1             | Sticky Note                | Documentation: Creation webhook input               | —                      | —                              | ## Creation – Webhook input                                  |
| Sticky Note2             | Sticky Note                | Documentation: ID generation and validation         | —                      | —                              | ## Creation – Generate & validate ID                        |
| Sticky Note4             | Sticky Note                | Documentation: Save, PDF generation, email          | —                      | —                              | ## Creation – Save, PDF & email                             |
| Sticky Note6             | Sticky Note                | Documentation: Check webhook input                   | —                      | —                              | ## Check – Webhook input                                    |
| Sticky Note7             | Sticky Note                | Documentation: Lookup logic                           | —                      | —                              | ## Check – Look up ID                                       |
| Sticky Note8             | Sticky Note                | Documentation: Response format                        | —                      | —                              | ## Check – Responses                                       |
| Sticky Note10            | Sticky Note                | Documentation: Overall workflow explanation          | —                      | —                              | ## Certifications – Creator & Checker                      |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Data Table**  
- Create a Data Table named `Certifications` with columns: `Name` (string), `Surname` (string), `CertificationID` (string).  
- This table stores the certificate records.

**Step 2: Setup Webhook for Certificate Creation**  
- Add a Webhook node named `Webhook_Creation`.  
- Configure HTTP Method: POST, Path: `certifications2`.  
- Response Mode: Last Node.

**Step 3: Generate Unique CertificationID**  
- Add a Code node `Generate_Certification_ID`.  
- Paste JavaScript code:  
  ```js
  const uniqueId =
    Date.now().toString(36).toUpperCase() +
    Math.random().toString(36).substring(2, 8).toUpperCase();
  return [{ id: uniqueId }];
  ```  
- Connect `Webhook_Creation` → `Generate_Certification_ID`.

**Step 4: Check ID Uniqueness in Data Table**  
- Add a Data Table node `Find_Certification_By_ID` with operation `get`.  
- Filter: `CertificationID` equals `={{ $json.id }}` (from Code node output).  
- Connect `Generate_Certification_ID` → `Find_Certification_By_ID`.

**Step 5: Conditional Check for ID Existence**  
- Add an If node `Certification_ID_Exists`.  
- Condition: Check if output JSON from `Find_Certification_By_ID` is empty (ID unique).  
- True branch: proceed to insert new record.  
- False branch: loop back to `Generate_Certification_ID` to generate a new ID.  
- Connect `Find_Certification_By_ID` → `Certification_ID_Exists`.

**Step 6: Insert New Certification Record**  
- Add a Data Table node `Insert_Certificaton`.  
- Operation: Insert.  
- Columns:  
  - `Name`: `={{ $('Webhook_Creation').item.json.headers.name }}`  
  - `Surname`: `={{ $('Webhook_Creation').item.json.headers.surname }}`  
  - `CertificationID`: `={{ $('Generate_Certification_ID').item.json.id }}`  
- Connect `Certification_ID_Exists` true branch → `Insert_Certificaton`.

**Step 7: Generate PDF Certificate**  
- Add PDF Generator API node `Generate a PDF document`.  
- Credentials: Set with your PDF Generator API account.  
- Template ID: Use your template (example `1560735`).  
- Data mapping:  
  ```json
  {
    "DueDate": "{{$now.toISODate()}}",
    "Candidate": "{{$('Webhook_Creation').item.json.headers.name}} {{$('Webhook_Creation').item.json.headers.surname}}",
    "CourseName": "{{$('Webhook_Creation').item.json.headers.course}}",
    "ID": "{{$('Generate_Certification_ID').item.json.id}}"
  }
  ```  
- Document output: file.  
- Connect `Insert_Certificaton` → `Generate a PDF document`.

**Step 8: Send Email with PDF Attachment**  
- Add Gmail node `Email_Certification`.  
- Credentials: Configure Gmail OAuth2 credentials.  
- Send To: `={{ $('Webhook_Creation').item.json.headers.email }}`  
- Subject: "Your certification is ready!"  
- Message: "Attached you find your new certification!"  
- Attachments: binary property `document.pdf` (from PDF node output).  
- Connect `Generate a PDF document` → `Email_Certification`.

**Step 9: Setup Webhook for Certificate Verification**  
- Add Webhook node `Webhook_Check`.  
- HTTP Method: POST, Path: `certificationscheck2`.  
- Response Mode: `responseNode`.

**Step 10: Lookup CertificationID**  
- Add Data Table node `Find_Certification_By_ID1`.  
- Operation: Get.  
- Filter: `CertificationID` equals `={{ $json.headers.id }}` (from webhook request).  
- Connect `Webhook_Check` → `Find_Certification_By_ID1`.

**Step 11: Check Existence of CertificationID**  
- Add If node `Certification_Exists`.  
- Condition: Check if `CertificationID` from Data Table equals header `id` from webhook.  
- True branch: go to respond with found data.  
- False branch: go to respond with not found.

**Step 12: Respond with Found Data**  
- Add Respond to Webhook node `Respond_Found`.  
- Response Body (JSON):  
  ```json
  {
    "ok": "true",
    "name": "{{ $('Find_Certification_By_ID1').item.json.Name }}",
    "surname": "{{ $('Find_Certification_By_ID1').item.json.Surname }}"
  }
  ```  
- Connect `Certification_Exists` true → `Respond_Found`.

**Step 13: Respond with Not Found**  
- Add Respond to Webhook node `Respond_NotFound`.  
- Response Body (JSON): `{ "ok": "false" }`  
- Connect `Certification_Exists` false → `Respond_NotFound`.

**Step 14: Deploy and Test**  
- Deploy both webhooks `/certifications2` and `/certificationscheck2`.  
- Test creation by POSTing learner data including `name`, `surname`, `email`, `course`.  
- Test verification by POSTing `id` (CertificationID) to `/certificationscheck2`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow manages the full lifecycle of digital training certificates, including creation, validation, PDF generation, and email delivery. It uses n8n Data Tables for persistent storage, PDF Generator API for templated certificate creation, and Gmail OAuth2 for sending emails.                                                                                                                                                                                                        | Workflow description and architecture overview.                                                                  |
| Setup steps include creating the Data Table with appropriate fields, configuring PDF Generator node with your account and HTML template (adapted for branding), connecting Gmail OAuth2 credentials, deploying the webhooks, and testing extensively before production use.                                                                                                                                                                                                              | Setup instructions from Sticky Note10.                                                                            |
| PDF Generator API template ID `1560735` is used as an example; users should configure their own template with branding and variable placeholders for `DueDate`, `Candidate`, `CourseName`, and `ID`.                                                                                                                                                                                                                                                                                             | PDF Generator API configuration details.                                                                          |
| Gmail OAuth2 credentials require client ID, secret, and OAuth consent configured in n8n credentials manager. Ensure proper scopes for sending emails.                                                                                                                                                                                                                                                                                                                                      | Gmail OAuth2 credential setup.                                                                                      |
| The workflow includes looping logic to ensure CertificationID uniqueness by regenerating the ID if a collision is detected. Infinite loop prevention should be considered if extremely high collision probability occurs.                                                                                                                                                                                                                                                                    | Logical flow for ID generation and validation.                                                                     |
| The verification endpoint returns a simple JSON structure usable by frontends or other services to confirm certificate validity, supporting integration in external applications.                                                                                                                                                                                                                                                                                                            | Verification response structure.                                                                                     |
| Video and blog resources about n8n and PDF Generator API integration may be found on n8n community forums and https://pdfgeneratorapi.com/blog/.                                                                                                                                                                                                                                                                                                                                             | External educational resources.                                                                                      |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.