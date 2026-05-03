Generate Business Documents with Google Sheets, CraftMyPDF, Drive & Gmail

https://n8nworkflows.xyz/workflows/generate-business-documents-with-google-sheets--craftmypdf--drive---gmail-5745


# Generate Business Documents with Google Sheets, CraftMyPDF, Drive & Gmail

### 1. Workflow Overview

This workflow automates the generation, storage, and distribution of personalized business documents in PDF format using Google Sheets, CraftMyPDF, Google Drive, and Gmail. Its main use case is bulk generation of contracts, job proposals, invoices, or any templated business documents by pulling data from a Google Sheet, creating PDFs through CraftMyPDF, uploading them to Google Drive, and emailing them to respective recipients.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Data Retrieval:** Triggered manually, reads employee or client data from a Google Sheet.
- **1.2 Iterative Processing:** Loops through each row of data to process documents individually.
- **1.3 PDF Document Creation:** Uses CraftMyPDF to generate a PDF document from the row data.
- **1.4 Validation & Retrieval:** Checks success status and obtains the generated PDF file link.
- **1.5 Storage & Distribution:** Uploads the PDF to Google Drive and sends it via Gmail.
- **1.6 Status Update & Loop Control:** Updates the Google Sheet row to mark completion and continues looping.
- **1.7 Merging Results:** Aggregates outputs from upload and email nodes to finalize the workflow.

Preliminary notes emphasize the necessity of using n8n self-hosted, setting up the Google Sheet and CraftMyPDF template properly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Retrieval

**Overview:**  
Starts the workflow via manual trigger and fetches employee data from a specified Google Sheet, filtering rows where the status column "DONE" is empty.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get employees (Google Sheets)

**Node Details:**  
- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual initiation  
  - Configuration: Default, no parameters  
  - Inputs: None  
  - Outputs: Data triggers next node  
  - Potential failures: None typical; manual start required

- **Get employees**  
  - Type: Google Sheets (Read)  
  - Role: Retrieves employee data rows to process  
  - Configuration: Reads from Sheet named “Foglio1” (gid=0) in a specific Google Sheet document  
  - Filters: Only rows where "DONE" column is empty (null)  
  - Credentials: Uses Google Sheets OAuth2 connection  
  - Inputs: Trigger node output  
  - Outputs: Array of employee data objects with all relevant fields (name, address, email, etc.)  
  - Potential failures: Google API auth errors, sheet ID/name errors, empty results if filter excludes all rows

---

#### 2.2 Iterative Processing

**Overview:**  
Splits the employee data array into individual items for sequential processing, facilitating stepwise document generation per employee.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)

**Node Details:**  
- Type: SplitInBatches  
- Role: Processes each employee record one at a time  
- Configuration: Default batch size (1) to process sequentially  
- Inputs: Array of employees from Get employees  
- Outputs: Single employee data per execution branch (main output 1), empty output branch (main output 0) for skipping conditions  
- Potential failures: Batch processing issues, expression errors in downstream nodes if data is malformed

---

#### 2.3 PDF Document Creation

**Overview:**  
Generates a personalized PDF document using CraftMyPDF API, injecting employee data into a predefined PDF template.

**Nodes Involved:**  
- Create agreement (CraftMyPDF)

**Node Details:**  
- Type: CraftMyPDF API  
- Role: Fill PDF template with dynamic data to create agreement document  
- Configuration:  
  - Template ID specified (unique to user’s CraftMyPDF template)  
  - Data fields mapped precisely from employee JSON properties (e.g., FIRST NAME, LAST NAME, ADDRESS)  
- Key expressions: Template data uses mustache-style interpolation `{{ $json["FIELD"] }}` for dynamic values  
- Credentials: CraftMyPDF API key  
- Inputs: Employee data from Loop Over Items output 1  
- Outputs: JSON containing PDF file URL and status  
- Potential failures: API auth errors, template ID invalid, data formatting issues, API rate limits

---

#### 2.4 Validation & Retrieval

**Overview:**  
Validates successful PDF creation by checking the API response status, then fetches the generated PDF file content via HTTP request.

**Nodes Involved:**  
- Success? (If)  
- Get agreement (HTTP Request)

**Node Details:**  
- **Success?**  
  - Type: If  
  - Role: Checks if CraftMyPDF returned status “success”  
  - Configuration: Compares `$json.status` equals "success"  
  - Inputs: Output of Create agreement  
  - Outputs: Passes to Get agreement if true; loops otherwise  
  - Potential failures: Expression evaluation errors, unexpected status values

- **Get agreement**  
  - Type: HTTP Request  
  - Role: Downloads the created PDF file from URL received from Create agreement  
  - Configuration: URL dynamically set to `{{ $('Create agreement').item.json.file }}`  
  - Inputs: Success? node output  
  - Outputs: Binary PDF data for next nodes  
  - Potential failures: Network timeout, invalid URL, file not found, HTTP errors (403, 404)

---

#### 2.5 Storage & Distribution

**Overview:**  
Uploads the PDF to a specified Google Drive folder and emails it as an attachment to the employee.

**Nodes Involved:**  
- Upload agreement (Google Drive)  
- Send email with PDF (Gmail)

**Node Details:**  
- **Upload agreement**  
  - Type: Google Drive (Upload)  
  - Role: Saves PDF file to Google Drive folder with employee name as filename  
  - Configuration:  
    - Filename dynamically constructed as `FIRST NAME-LAST NAME.pdf`  
    - Folder ID preconfigured (specific Drive folder)  
  - Credentials: Google Drive OAuth2  
  - Inputs: Binary PDF from Get agreement  
  - Outputs: Metadata of uploaded file  
  - Potential failures: Drive API auth errors, permission denied, storage quota exceeded

- **Send email with PDF**  
  - Type: Gmail (Send Email)  
  - Role: Sends personalized email with PDF attachment to employee email address  
  - Configuration:  
    - Recipient set to employee EMAIL field  
    - Subject "Job offer"  
    - Message body with dynamic first and last name interpolation  
    - Attachment: binary PDF file from Get agreement  
  - Credentials: Gmail OAuth2  
  - Inputs: Binary PDF, employee data  
  - Outputs: Email send confirmation  
  - Potential failures: Gmail API auth errors, quota limits, invalid recipient email, attachment size limits

---

#### 2.6 Status Update & Loop Control

**Overview:**  
Marks the processed row in Google Sheets as done by updating the “DONE” column, then loops back to process the next employee.

**Nodes Involved:**  
- Update row (Google Sheets)  
- Loop Over Items (SplitInBatches) [loop back]

**Node Details:**  
- **Update row**  
  - Type: Google Sheets (Update)  
  - Role: Updates the current row’s “DONE” column to “x” to mark completion  
  - Configuration:  
    - Matches row by “row_number” field from Loop Over Items  
    - Sets DONE = “x”  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Employee data and row number from Loop Over Items  
  - Outputs: Confirmation of update  
  - Potential failures: Row not found, permission denied, Google API errors

- **Loop Over Items**  
  - Receives updated data and continues processing next item until all rows are done.

---

#### 2.7 Merging Results

**Overview:**  
Combines outputs from both “Upload agreement” and “Send email with PDF” nodes into a single stream to finalize processing.

**Nodes Involved:**  
- Merge

**Node Details:**  
- Type: Merge  
- Role: Joins parallel outputs from upload and email nodes  
- Configuration: Default merge mode (append or combine)  
- Inputs: Two main inputs, one from Upload agreement, one from Send email with PDF  
- Outputs: Final aggregated output  
- Potential failures: Mismatched data streams, missing inputs

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                             | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                                                                    |
|----------------------------|-----------------------------|--------------------------------------------|--------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Entry point to start workflow manually     | None                           | Get employees                |                                                                                                                                                |
| Get employees              | Google Sheets (Read)         | Reads employee data from Google Sheet      | When clicking ‘Execute workflow’ | Loop Over Items              |                                                                                                                                                |
| Loop Over Items            | SplitInBatches               | Processes each employee record iteratively | Get employees                  | Create agreement, (empty output) |                                                                                                                                                |
| Create agreement           | CraftMyPDF API               | Generates personalized PDF from template   | Loop Over Items (output 1)     | Success?                    |                                                                                                                                                |
| Success?                  | If                          | Checks if PDF creation succeeded            | Create agreement               | Get agreement (if success)   |                                                                                                                                                |
| Get agreement             | HTTP Request                 | Downloads generated PDF file                 | Success?                      | Upload agreement, Send email with PDF |                                                                                                                                                |
| Upload agreement          | Google Drive (Upload)         | Uploads PDF file to Google Drive             | Get agreement                 | Merge                       |                                                                                                                                                |
| Send email with PDF       | Gmail (Send Email)            | Sends email with PDF attachment              | Get agreement                 | Merge                       |                                                                                                                                                |
| Merge                    | Merge                        | Combines upload and email outputs            | Upload agreement, Send email with PDF | Update row                  |                                                                                                                                                |
| Update row               | Google Sheets (Update)        | Marks row as done in Google Sheet            | Merge                        | Loop Over Items              |                                                                                                                                                |
| Sticky Note              | Sticky Note                  | Notes on preliminary setup and usage        | None                         | None                        | ## PRELIMINARY STEPS - Only works with the self-hosted version of n8n - Clone this [Sheet](https://docs.google.com/spreadsheets/d/1YQPuoEubRHJepRKdquks69Iqf2XdGVKfpWOdYwk3RMg/edit?usp=sharing) - Create an account on [CraftMyPDF](https://app.craftmypdf.com/) - Create a new PDF template - Get the Template ID and insert it in the "Create aggreement" node |
| Sticky Note1             | Sticky Note                  | Workflow purpose and summary                 | None                         | None                        | ## Auto-Generate Business Documents in PDF This workflow allows you to generate contracts in bulk (job proposals, general documents, pay slips, invoices, contracts, etc.) in PDF format, starting from a Google Sheet containing the data to be inserted. After creating a PDF template (using CraftMyPDF), this workflow manages the entire process, from filling out the PDF files, to sending emails with attachments, to archiving them on Google Drive (or an equivalent system), in a fully automated way. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named “When clicking ‘Execute workflow’”. No configuration needed.

2. **Set Up Google Sheets Read Node:**  
   - Add Google Sheets node named “Get employees” configured to:  
     - Operation: Read rows  
     - Document ID: Your Google Sheet’s ID (e.g., `1YQPuoEubRHJepRKdquks69Iqf2XdGVKfpWOdYwk3RMg`)  
     - Sheet Name: Use the first sheet or gid=0 (e.g., “Foglio1”)  
     - Filters: Only retrieve rows where column “DONE” is empty (null)  
   - Connect Manual Trigger output to this node.  
   - Use Google Sheets OAuth2 credentials.

3. **Add SplitInBatches Node:**  
   - Add SplitInBatches node named “Loop Over Items”  
   - Set batch size to 1 (default) to process one employee at a time  
   - Connect “Get employees” main output to this node.

4. **Add CraftMyPDF Node for PDF Generation:**  
   - Add CraftMyPDF node named “Create agreement”  
   - Operation: Create PDF from template  
   - Insert your CraftMyPDF Template ID (from your account)  
   - Map data fields dynamically based on employee data, for example:  
     ```
     {
       "first_name": "{{ $json[\"FIRST NAME\"] }}",
       "last_name": "{{ $json[\"LAST NAME\"] }}",
       "address_street1": "{{ $json.ADDRESS }}",
       "address_city": "{{ $json.CITY }}",
       "address_state": "{{ $json.STATE }}",
       "address_postal": "{{ $json[\"ZIP CODE\"] }}",
       "address_country": "{{ $json.COUNTRY }}",
       "email": "{{ $json.EMAIL }}",
       "phone_number": "{{ $json.PHONE }}",
       "position": "{{ $json.POSITION }}"
     }
     ```  
   - Connect SplitInBatches output (main output 1) to this node.  
   - Use CraftMyPDF API credentials.

5. **Add If Node to Check Success:**  
   - Add If node named “Success?”  
   - Condition: Check if `{{$json.status}}` equals “success”  
   - Connect “Create agreement” output to this node.

6. **Add HTTP Request Node to Download PDF:**  
   - Add HTTP Request node named “Get agreement”  
   - Method: GET  
   - URL: `={{ $('Create agreement').item.json.file }}`  
   - Connect “Success?” true output to this node.

7. **Add Google Drive Upload Node:**  
   - Add Google Drive node named “Upload agreement”  
   - Operation: Upload file  
   - File Name: `={{ $('Get employees').item.json["FIRST NAME"] + '-' + $('Get employees').item.json["LAST NAME"] + '.pdf' }}`  
   - Folder: Select your target Google Drive folder by ID  
   - Connect “Get agreement” output to this node.  
   - Use Google Drive OAuth2 credentials.

8. **Add Gmail Send Email Node:**  
   - Add Gmail node named “Send email with PDF”  
   - To: `={{ $('Get employees').item.json.EMAIL }}`  
   - Subject: “Job offer” (or your subject)  
   - Message: Use HTML or plain text, e.g.,  
     ```
     Hi {{ $('Get employees').item.json['FIRST NAME'] }} {{ $('Get employees').item.json['LAST NAME'] }},<br>
     Attached is our job proposal<br><br>
     Best,<br>
     [Company name]
     ```  
   - Attachments: Use binary data from “Get agreement” node  
   - Connect “Get agreement” output to this node.  
   - Use Gmail OAuth2 credentials.

9. **Add Merge Node:**  
   - Add Merge node named “Merge”  
   - Merge mode: Append (default)  
   - Connect “Upload agreement” and “Send email with PDF” outputs as inputs to Merge.

10. **Add Google Sheets Update Node:**  
    - Add Google Sheets node named “Update row”  
    - Operation: Update row  
    - Document ID and Sheet Name: same as “Get employees”  
    - Matching column: “row_number” from current batch item  
    - Update column “DONE” to “x”  
    - Connect “Merge” output to this node.  
    - Use Google Sheets OAuth2 credentials.

11. **Loop Back to SplitInBatches:**  
    - Connect “Update row” output back to “Loop Over Items” input to continue processing next row until completion.

12. **Add Sticky Notes (Optional):**  
    - Add notes with instructions for setup, including:  
      - Cloning the Google Sheet template  
      - Creating CraftMyPDF account and template  
      - Setting Template ID in “Create agreement” node  
      - Workflow description and purpose

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires n8n self-hosted version as external API credentials and certain OAuth scopes are involved.                                                                                                                                                                                                                                                                | -                                                                                                                                    |
| Clone this Google Sheet template for employee/job data: https://docs.google.com/spreadsheets/d/1YQPuoEubRHJepRKdquks69Iqf2XdGVKfpWOdYwk3RMg/edit?usp=sharing                                                                                                                                                                                                                   | Google Sheets Template                                                                                                               |
| Create an account and PDF template at CraftMyPDF: https://app.craftmypdf.com/                                                                                                                                                                                                                                                                                                      | CraftMyPDF official site                                                                                                            |
| Insert your CraftMyPDF Template ID into the “Create agreement” node to enable PDF generation.                                                                                                                                                                                                                                                                                      | Configuration detail                                                                                                                |
| Workflow allows bulk generation of contracts, invoices, proposals, etc., automating document creation, storage, and emailing seamlessly.                                                                                                                                                                                                                                         | Workflow purpose                                                                                                                    |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a powerful integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.