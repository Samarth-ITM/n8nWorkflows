Automated New Hire Appointment Letters with Google Docs, HR Approval & Gmail

https://n8nworkflows.xyz/workflows/automated-new-hire-appointment-letters-with-google-docs--hr-approval---gmail-5879


# Automated New Hire Appointment Letters with Google Docs, HR Approval & Gmail

---

### 1. Workflow Overview

This workflow automates the generation, approval, and distribution of new hire appointment letters using Google Docs, Google Drive, and Gmail, with a Human-In-The-Loop approval step involving an HR manager. It targets HR departments aiming to streamline appointment letter creation and approval by integrating form input, document templating, PDF generation, manual approval, and email notifications.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Captures candidate and offer details via a form submission trigger.
- **1.2 Document Preparation:** Creates a personalized appointment letter from a Google Docs template, replaces placeholders with form data, converts it to PDF, and stores it on Google Drive.
- **1.3 HR Approval:** Sends an approval request email to the HR manager containing the appointment letter link or details, and waits for approval or rejection.
- **1.4 Candidate Notification:** Upon HR approval, downloads the approved PDF and emails it to the candidate with the appointment letter attached.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow by capturing candidate and offer details submitted via an n8n form trigger titled "Generate Appointment Letter." The collected data drives the entire downstream document generation and communication process.

**Nodes Involved:**  
- On form submission  
- Sticky Note (instructional)

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger (Webhook-based)  
  - *Role:* Entry point receiving structured input fields: Candidate Name, Position Name, Fixed CTC, Joining Date, To be signed by Date, Candidate email.  
  - *Configuration:* Triggered on form submit with explicit form fields configured to ensure data structure.  
  - *Expressions:* Uses standard field labels to store values in JSON for downstream use.  
  - *Connections:* Outputs to "Create a candidate copy of the appointment letter template."  
  - *Edge Cases:* Missing or malformed form data could cause errors downstream; validation should be enforced on the form itself.  
  - *Notes:* Sticky Note nearby advises that these form values will update the appointment letter template.

- **Sticky Note**  
  - *Content:* "Use the form fields to update the Appointment letter template and create a candidate specific appointment letter in PDF format on the Google drive."  
  - *Role:* Documentation aid only.

---

#### 1.2 Document Preparation

**Overview:**  
This block creates a personalized copy of a Google Docs appointment letter template, replaces placeholders with candidate-specific data, converts the document to PDF, and uploads the PDF to a designated Google Drive folder.

**Nodes Involved:**  
- Create a candidate copy of the appointment letter template  
- Sticky Note1  
- Update the candidate appointment letter with offer details  
- Download the appointment letter as a PDF  
- Upload the PDF to Google drive

**Node Details:**  

- **Create a candidate copy of the appointment letter template**  
  - *Type:* Google Drive - Copy File  
  - *Role:* Copies a master Google Docs template file and renames it using the candidate’s name to create a unique document.  
  - *Configuration:*  
    - File ID points to a fixed Google Docs template.  
    - Name expression: `"Appointment Letter - {{ $json['Candidate Name'] }}"`  
    - Copy does not require writer permission changes.  
  - *Input:* Receives form submission data.  
  - *Output:* Passes copied document metadata, including new document ID.  
  - *Edge Cases:* Permission errors if OAuth token lacks copy rights; template file missing or inaccessible.  

- **Sticky Note1**  
  - *Content:* "Create a candidate specific version of the appointment letter and create a PDF version of it. Store this on the Google Drive."  
  - *Role:* Instructional context for this block.

- **Update the candidate appointment letter with offer details**  
  - *Type:* Google Docs - Update Document  
  - *Role:* Performs multiple find-and-replace actions in the copied Google Docs file to insert candidate-specific details.  
  - *Configuration:*  
    - Target document URL is dynamically set to the copied document ID.  
    - Replaces placeholders such as `[Candidate Name]`, `[Position Name]`, `[Fixed CTC]`, `[Joining Date]`, `[To be signed by Date]`, and `[Date]` with corresponding form data or current date formatted as MM-dd-yyyy.  
  - *Input:* Document copy metadata from previous node.  
  - *Output:* Updated document metadata including document ID for next step.  
  - *Edge Cases:* Failure if document ID invalid or if placeholder texts missing in template; expression evaluation errors.  

- **Download the appointment letter as a PDF**  
  - *Type:* Google Drive - Download File  
  - *Role:* Downloads the updated Google Docs document and converts it to PDF binary data.  
  - *Configuration:*  
    - File ID dynamically set from updated document metadata.  
    - Conversion set from Google Docs format to PDF MIME type.  
    - Binary data stored under property "data."  
  - *Input:* Output of document update node.  
  - *Output:* Binary PDF data for upload.  
  - *Edge Cases:* Download or conversion failures; file permission errors.  

- **Upload the PDF to Google drive**  
  - *Type:* Google Drive - Upload File  
  - *Role:* Uploads the PDF binary file to a specific Google Drive folder designated for appointment letters.  
  - *Configuration:*  
    - Filename preserved from candidate copy name.  
    - Drive ID set to "My Drive" (default user drive).  
    - Folder ID fixed to a specific folder (e.g., "Appointment Letter").  
    - Binary data implicitly passed in the workflow.  
  - *Input:* PDF binary from previous node.  
  - *Output:* Metadata of uploaded PDF file including its Google Drive file ID.  
  - *Edge Cases:* Upload errors due to insufficient permissions or quota limits; folder ID invalid.  

---

#### 1.3 HR Approval

**Overview:**  
Introduces a Human-In-The-Loop step by sending an email to the HR manager requesting approval of the generated appointment letter. The workflow waits asynchronously for the HR manager's response before proceeding.

**Nodes Involved:**  
- Send message and wait for response  
- Sticky Note2  
- If (approval check)

**Node Details:**  

- **Send message and wait for response**  
  - *Type:* Gmail - Send And Wait for Response  
  - *Role:* Sends an email to the HR manager with appointment letter details and waits for an approval or rejection response asynchronously.  
  - *Configuration:*  
    - Recipient: hr.manager@gmail.com (hardcoded).  
    - Subject: "Approval required - [filename]" dynamically set to the uploaded PDF name.  
    - Body: A polite message requesting review and approval for the candidate's appointment letter.  
    - Approval Options: Set to "double" approval type (likely meaning approval or rejection).  
  - *Input:* Uploaded PDF metadata.  
  - *Output:* JSON data containing approval status.  
  - *Edge Cases:* Email sending failures, OAuth token expiration, HR manager non-response causing workflow delays or timeouts.  

- **Sticky Note2**  
  - *Content:* "Human-In-The-Loop step - Send an email to the HR manager requesting to review the appointment letter and 'Approve' or 'Reject'."  
  - *Role:* Documentation for this approval step.

- **If**  
  - *Type:* Conditional (If)  
  - *Role:* Checks if HR approved the appointment letter to determine next action.  
  - *Configuration:*  
    - Condition tests if `$json.data.approved` is true (boolean).  
  - *Input:* Approval response from previous node.  
  - *Output:* Routes workflow: if true, proceeds to candidate notification; if false or no approval, workflow ends or could be extended.  
  - *Edge Cases:* Missing or malformed approval data; false negatives if approval field absent.

---

#### 1.4 Candidate Notification

**Overview:**  
Upon HR approval, downloads the approved PDF from Google Drive and sends it as an email attachment to the candidate, notifying them with joining instructions.

**Nodes Involved:**  
- Download file  
- Send a message  
- Sticky Note3

**Node Details:**  

- **Download file**  
  - *Type:* Google Drive - Download File  
  - *Role:* Downloads the approved appointment letter PDF file from Google Drive to attach it to the candidate's email.  
  - *Configuration:*  
    - File ID dynamically taken from the uploaded PDF metadata.  
    - No conversion applied; binary data output is PDF.  
  - *Input:* Routed from the If node upon approval.  
  - *Output:* Binary PDF data.  
  - *Edge Cases:* Download failure or permission issues; file may have been deleted or moved.

- **Send a message**  
  - *Type:* Gmail - Send Email  
  - *Role:* Sends the appointment letter PDF as an email attachment to the candidate's email address.  
  - *Configuration:*  
    - Recipient: dynamically set to candidate email captured at form submission.  
    - Subject: "Appointment Letter" (static).  
    - Body: Congratulatory message including candidate name and "To be signed by" date.  
    - Attachment: PDF binary from the previous download node.  
  - *Input:* PDF binary plus candidate email data.  
  - *Output:* Email sent confirmation.  
  - *Edge Cases:* Email sending failure, invalid candidate email, attachment missing or corrupt.  

- **Sticky Note3**  
  - *Content:* "If the HR manager approves, send an email to the candidate with the appointment letter as a PDF attachment."  
  - *Role:* Documentation for final notification step.

---

### 3. Summary Table

| Node Name                                 | Node Type               | Functional Role                                   | Input Node(s)                              | Output Node(s)                         | Sticky Note                                                                                         |
|-------------------------------------------|-------------------------|-------------------------------------------------|--------------------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------|
| On form submission                        | Form Trigger            | Captures candidate and offer details             | None                                       | Create a candidate copy of the appointment letter template | Use the form fields to update the Appointment letter template and create a candidate specific appointment letter in PDF format on the Google drive |
| Sticky Note                              | Sticky Note             | Instructional                                    | None                                       | None                                 | Use the form fields to update the Appointment letter template and create a candidate specific appointment letter in PDF format on the Google drive |
| Create a candidate copy of the appointment letter template | Google Drive - Copy File | Copies master template and renames it per candidate | On form submission                         | Update the candidate appointment letter with offer details | Create a candidate specific version of the appointment letter and create a PDF version of it. Store this on the Google Drive |
| Sticky Note1                            | Sticky Note             | Instructional                                    | None                                       | None                                 | Create a candidate specific version of the appointment letter and create a PDF version of it. Store this on the Google Drive |
| Update the candidate appointment letter with offer details | Google Docs - Update Document | Replaces placeholders with candidate data       | Create a candidate copy of the appointment letter template | Download the appointment letter as a PDF | Create a candidate specific version of the appointment letter and create a PDF version of it. Store this on the Google Drive |
| Download the appointment letter as a PDF | Google Drive - Download File | Converts Google Doc to PDF and downloads         | Update the candidate appointment letter with offer details | Upload the PDF to Google drive        | Create a candidate specific version of the appointment letter and create a PDF version of it. Store this on the Google Drive |
| Upload the PDF to Google drive           | Google Drive - Upload File | Uploads PDF file to designated folder             | Download the appointment letter as a PDF   | Send message and wait for response    | Create a candidate specific version of the appointment letter and create a PDF version of it. Store this on the Google Drive |
| Send message and wait for response       | Gmail - Send & Wait     | Sends approval request email to HR manager       | Upload the PDF to Google drive             | If                                   | Human-In-The-Loop step - Send an email to the HR manager requesting to review the appointment letter and "Approve" or "Reject" |
| Sticky Note2                            | Sticky Note             | Instructional                                    | None                                       | None                                 | Human-In-The-Loop step - Send an email to the HR manager requesting to review the appointment letter and "Approve" or "Reject" |
| If                                       | If                      | Checks HR approval status                         | Send message and wait for response         | Download file                        |                                                                                                   |
| Download file                            | Google Drive - Download File | Downloads approved PDF from Google Drive          | If                                          | Send a message                       | If the HR manager approves, send an email to the candidate with the appointment letter as a PDF attachment |
| Send a message                          | Gmail - Send             | Sends appointment letter PDF to candidate email | Download file                              | None                                 | If the HR manager approves, send an email to the candidate with the appointment letter as a PDF attachment |
| Sticky Note3                            | Sticky Note             | Instructional                                    | None                                       | None                                 | If the HR manager approves, send an email to the candidate with the appointment letter as a PDF attachment |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Configure form title as "Generate Appointment Letter."  
   - Add form fields (all text): Candidate Name, Position Name, Fixed CTC, Joining Date, To be signed by Date, Candidate email.  
   - This node acts as the webhook entry point.

2. **Create Google Drive Node ("Create a candidate copy of the appointment letter template"):**  
   - Type: Google Drive (Copy File)  
   - Set operation to "copy."  
   - Provide the file ID of the master Google Docs appointment letter template.  
   - Set the new file name using expression: `Appointment Letter - {{ $json["Candidate Name"] }}`.  
   - Use valid Google Drive OAuth2 credentials with copy permissions.

3. **Create Google Docs Node ("Update the candidate appointment letter with offer details"):**  
   - Type: Google Docs (Update Document)  
   - Operation: Update  
   - Document URL: Use the copied document’s ID from previous node output.  
   - Set multiple "replaceAll" actions to substitute placeholders:  
     - `[Candidate Name]` → `{{ $json["Candidate Name"] }}`  
     - `[Position Name]` → `{{ $json["Position Name"] }}`  
     - `[Fixed CTC]` → `{{ $json["Fixed CTC"] }}`  
     - `[Joining Date]` → `{{ $json["Joining Date"] }}`  
     - `[To be signed by Date]` → `{{ $json["To be signed by Date"] }}`  
     - `[Date]` → Current date formatted as MM-dd-yyyy using `$now.toFormat('MM-dd-yyyy')`.  
   - Use Google Docs OAuth2 credentials.

4. **Create Google Drive Node ("Download the appointment letter as a PDF"):**  
   - Type: Google Drive (Download File)  
   - Operation: Download  
   - File ID: Use the updated Google Docs document ID.  
   - Enable Google file conversion: Docs to PDF (`application/pdf`).  
   - Set binary property name to "data."  
   - Use Google Drive OAuth2 credentials.

5. **Create Google Drive Node ("Upload the PDF to Google drive"):**  
   - Type: Google Drive (Upload File)  
   - Operation: Upload  
   - Filename: Use the same name as the copied appointment letter document.  
   - Drive ID: Set to "My Drive."  
   - Folder ID: Set to the folder designated for appointment letters (e.g., a specific folder ID in Google Drive).  
   - Use Google Drive OAuth2 credentials.

6. **Create Gmail Node ("Send message and wait for response"):**  
   - Type: Gmail (Send and Wait)  
   - Send To: HR manager’s email (e.g., hr.manager@gmail.com).  
   - Subject: `Approval required - {{ $json["name"] }}` (dynamic filename).  
   - Message body: Inform HR to review and approve the appointment letter for the candidate.  
   - Set approval options: type "double" for approval or rejection.  
   - Use Gmail OAuth2 credentials.

7. **Create If Node ("If"):**  
   - Type: If condition node  
   - Condition: Check if `$json.data.approved === true`.  
   - This node branches workflow flow based on approval.

8. **Create Google Drive Node ("Download file"):**  
   - Type: Google Drive (Download File)  
   - Operation: Download  
   - File ID: Use the uploaded PDF file ID from "Upload the PDF to Google drive."  
   - Use Google Drive OAuth2 credentials.

9. **Create Gmail Node ("Send a message"):**  
   - Type: Gmail (Send)  
   - Send To: Candidate email from form submission.  
   - Subject: "Appointment Letter."  
   - Message body: Congratulate candidate and mention signing deadline date.  
   - Attach the PDF binary downloaded from previous node.  
   - Use Gmail OAuth2 credentials.

10. **Connect Nodes in This Order:**  
    - On form submission → Create a candidate copy of the appointment letter template → Update the candidate appointment letter with offer details → Download the appointment letter as a PDF → Upload the PDF to Google drive → Send message and wait for response → If → Download file → Send a message.

11. **Add Sticky Notes:**  
    - To document instructions and logical groupings as per the original workflow.

12. **Credential Setup:**  
    - Google Drive OAuth2 with read/write and file copy permissions.  
    - Google Docs OAuth2 with document update permissions.  
    - Gmail OAuth2 with send and read permissions for approval handling.

13. **Validate Permissions and Folder IDs:**  
    - Ensure the Google Drive folder for storing PDFs exists and is accessible.  
    - Verify OAuth credentials are valid and authorized for all required APIs.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Use a master Google Docs template with clearly marked placeholders matching the exact text replaced. | Ensures Google Docs update node works reliably.                                                |
| The Gmail "Send and Wait" node enables Human-In-The-Loop approval workflows via email interaction.   | n8n documentation on Gmail node approval options for advanced email workflows.                  |
| Folder ID and Template file ID are critical constants and must be accessible to the OAuth user.      | Obtain from Google Drive URLs or API; incorrect IDs will cause node failures.                   |
| Date formatting uses n8n’s `$now.toFormat()` method for dynamic timestamping.                         | Documentation: https://docs.n8n.io/nodes/expressions/                                           |
| Hardcoded HR manager email can be parameterized for multi-user or environment deployment flexibility.| Useful for scaling or adapting workflow to different HR managers or teams.                      |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created in n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---