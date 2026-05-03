Generate and email event e-tickets with QR codes using Google Workspace

https://n8nworkflows.xyz/workflows/generate-and-email-event-e-tickets-with-qr-codes-using-google-workspace-14500


# Generate and email event e-tickets with QR codes using Google Workspace

## 1. Workflow Overview

This workflow automatically generates personalized event e-tickets as Google Docs, inserts a QR code tied to each registrant’s ticket ID, exports the result as a PDF, stores that PDF in Google Drive, and emails the participant a download link.

It is designed for event registration flows where participant data lands in Google Sheets, typically from a Google Form or another upstream intake process. Common use cases include marathons, workshops, webinars, seminars, and any event where each participant needs a unique pass or scannable check-in token.

### 1.1 Input Reception
The workflow starts when a new row is detected in a specific Google Sheets worksheet. That row contains participant and ticket information such as name, email, and ticket ID.

### 1.2 Ticket Document Creation
A Google Docs template is duplicated for the registrant. The copied document is then updated by replacing template placeholders with actual participant data.

### 1.3 QR Code Injection
A QR code image is generated dynamically using an external image URL and inserted into the copied Google Doc through the Google Docs API batchUpdate endpoint.

### 1.4 PDF Export and Storage
The completed Google Doc is exported to PDF and uploaded to a designated Google Drive folder for distribution and retention.

### 1.5 Email Dispatch
A branded HTML email is sent to the participant using SMTP, including a download button pointing to the stored PDF file.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block watches a Google Sheets worksheet for new incoming registration rows. It serves as the workflow entry point and provides all participant-specific variables used by downstream nodes.

**Nodes Involved:**  
- Google Sheets Trigger

### Node Details

#### Google Sheets Trigger
- **Type and technical role:** `n8n-nodes-base.googleSheetsTrigger`  
  Polling trigger node that monitors a Google Sheets worksheet for new rows.
- **Configuration choices:**  
  - Polls every minute.
  - Watches a specific spreadsheet and worksheet:
    - Spreadsheet: `Salinan dari QR Codes`
    - Sheet/tab: `QR Code Data`
- **Key expressions or variables used:**  
  Downstream nodes reference fields from this trigger, including:
  - `$('Google Sheets Trigger').item.json.TIKET`
  - `$('Google Sheets Trigger').item.json.NAMA`
  - `$('Google Sheets Trigger').item.json.Email`
- **Input and output connections:**  
  - Input: none, this is the entry point
  - Output: `Make Copy of Template`
- **Version-specific requirements:**  
  - Uses node type version `1`
  - Requires valid Google Sheets Trigger OAuth2 credentials
- **Edge cases or potential failure types:**  
  - OAuth token expiration or revoked permissions
  - Wrong spreadsheet or worksheet selection
  - Missing expected columns like `TIKET`, `NAMA`, or `Email`
  - Polling delay because this is not instant webhook-based triggering
  - Duplicate processing depending on how rows are added/updated in the sheet
- **Sub-workflow reference:**  
  None

---

## 2.2 Ticket Document Creation

**Overview:**  
This block creates a personalized Google Doc from a master template and replaces at least one placeholder with participant-specific data. It is the core document assembly stage before QR image insertion.

**Nodes Involved:**  
- Make Copy of Template
- Change Custom Variables

### Node Details

#### Make Copy of Template
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Copies an existing Google Docs file in Google Drive to create a new document for the participant.
- **Configuration choices:**  
  - Operation: `copy`
  - Source file: a Google Docs template named `QR Test`
  - New file name expression: `eTicket for {{ $json.TIKET }}`
  - `executeOnce` is enabled
- **Key expressions or variables used:**  
  - New file name uses current item field: `$json.TIKET`
- **Input and output connections:**  
  - Input: `Google Sheets Trigger`
  - Output: `Change Custom Variables`
- **Version-specific requirements:**  
  - Uses Google Drive node version `3`
  - Requires Google Drive OAuth2 credentials with access to the template file
- **Edge cases or potential failure types:**  
  - Template file no longer exists or access is lost
  - Filename expression fails if `TIKET` is absent
  - `executeOnce` can be problematic in multi-item executions if more than one item enters in a single run
- **Sub-workflow reference:**  
  None

#### Change Custom Variables
- **Type and technical role:** `n8n-nodes-base.googleDocs`  
  Updates the copied Google Doc by replacing placeholder text.
- **Configuration choices:**  
  - Operation: `update`
  - Target document URL/id comes from the copied file: `={{ $json.id }}`
  - One replace action is defined:
    - Replace `{kode_qr}` with the ticket ID from the trigger row
- **Key expressions or variables used:**  
  - `$('Google Sheets Trigger').item.json.TIKET`
  - `{{ $json.id }}` from the previous Google Drive copy result
- **Input and output connections:**  
  - Input: `Make Copy of Template`
  - Output: `Insert image`
- **Version-specific requirements:**  
  - Uses Google Docs node version `2`
  - Requires Google Docs OAuth2 credentials
- **Edge cases or potential failure types:**  
  - Placeholder `{kode_qr}` not present in the template
  - Document URL/id mismatch if previous node output changes
  - Failure if the copied file is not recognized as a Google Doc
  - Missing source fields from the trigger row
- **Sub-workflow reference:**  
  None

---

## 2.3 QR Code Injection

**Overview:**  
This block inserts a QR code image into the personalized Google Doc using the Google Docs API directly. It complements the text replacement step by embedding a machine-readable check-in asset into the ticket.

**Nodes Involved:**  
- Insert image

### Node Details

#### Insert image
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a raw authenticated HTTP request to the Google Docs API `batchUpdate` endpoint to insert an inline image into the document.
- **Configuration choices:**  
  - Method: `POST`
  - URL pattern: `https://docs.googleapis.com/v1/documents/{{$json.documentId}}:batchUpdate`
  - Authentication: predefined credential type using Google Docs OAuth2
  - Body contains a `requests` array with one `insertInlineImage` action
  - The image URI is generated by QuickChart:
    `https://quickchart.io/qr?format=png&text=$('Google Sheets Trigger').item.json.TIKET`
  - Image is inserted at document index `27`
  - Response format is configured as full response in text mode
- **Key expressions or variables used:**  
  - `$json.documentId` from `Change Custom Variables`
  - `$('Google Sheets Trigger').item.json.TIKET` embedded in the QR image URL
- **Input and output connections:**  
  - Input: `Change Custom Variables`
  - Output: `Generate PDF`
- **Version-specific requirements:**  
  - Uses HTTP Request node version `3`
  - Requires Google Docs OAuth2 credential authorized for Docs API calls
- **Edge cases or potential failure types:**  
  - The insertion index `27` is hardcoded and may be invalid if the template structure changes
  - If the QuickChart service is unavailable, the image may fail to render or insert
  - Incorrect expression composition could produce a broken image URL
  - Google Docs batchUpdate may fail if the document is not fully ready immediately after replacement
  - API permission scopes may be insufficient for document editing
- **Sub-workflow reference:**  
  None

---

## 2.4 PDF Export and Storage

**Overview:**  
This block converts the completed Google Doc into PDF format and saves it into a target Google Drive folder. It turns the editable working document into a distributable final artifact.

**Nodes Involved:**  
- Generate PDF
- Add PDF To Drive

### Node Details

#### Generate PDF
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads a Google Docs file while converting it to PDF.
- **Configuration choices:**  
  - Operation: `download`
  - File ID comes from `Change Custom Variables.documentId`
  - Google file conversion enabled:
    - Docs to format: `application/pdf`
  - Output binary property name: `data`
- **Key expressions or variables used:**  
  - `={{ $('Change Custom Variables').item.json.documentId }}`
- **Input and output connections:**  
  - Input: `Insert image`
  - Output: `Add PDF To Drive`
- **Version-specific requirements:**  
  - Uses Google Drive node version `3`
  - Requires Google Drive OAuth2 credentials with file download/export access
- **Edge cases or potential failure types:**  
  - If the document update or image insertion has not fully propagated, exported PDF may miss recent content
  - Export can fail if file ID is invalid
  - Binary data issues if `data` property is overwritten later
- **Sub-workflow reference:**  
  None

#### Add PDF To Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the generated PDF binary file to a designated Google Drive folder.
- **Configuration choices:**  
  - Target drive: `My Drive`
  - Destination folder: `eTicket`
  - File name uses the copied document name from the template copy node
- **Key expressions or variables used:**  
  - `={{ $('Make Copy of Template').item.json.name }}`
- **Input and output connections:**  
  - Input: `Generate PDF`
  - Output: `Send an Email`
- **Version-specific requirements:**  
  - Uses Google Drive node version `3`
  - Requires upload permission to the chosen Drive folder
- **Edge cases or potential failure types:**  
  - If the node is not explicitly configured to use the incoming binary property, upload may fail depending on default node behavior and n8n version
  - File naming collisions may occur if the same ticket is generated more than once
  - Drive folder permissions may prevent upload
  - `webContentLink` may not be usable by recipients unless sharing permissions are configured
- **Sub-workflow reference:**  
  None

---

## 2.5 Email Dispatch

**Overview:**  
This block sends a formatted HTML email to the participant after the PDF has been stored in Google Drive. The email contains participant-specific content and a direct download button to the uploaded file.

**Nodes Involved:**  
- Send an Email

### Node Details

#### Send an Email
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends an outbound email through SMTP.
- **Configuration choices:**  
  - Recipient: `={{ $('Google Sheets Trigger').item.json.Email }}`
  - Sender: `user@example.com`
  - Subject: `e-Ticket Resmi Anda untuk Event [Nama Event] 🎟️`
  - HTML body includes:
    - Greeting with participant name
    - Ticket ID
    - Download button using `{{ $json.webContentLink }}`
- **Key expressions or variables used:**  
  - `$('Google Sheets Trigger').item.json.NAMA`
  - `$('Google Sheets Trigger').item.json.TIKET`
  - `$('Google Sheets Trigger').item.json.Email`
  - `$json.webContentLink` from `Add PDF To Drive`
- **Input and output connections:**  
  - Input: `Add PDF To Drive`
  - Output: none
- **Version-specific requirements:**  
  - Uses Email Send node version `2.1`
  - Requires valid SMTP credentials
- **Edge cases or potential failure types:**  
  - SMTP authentication failure
  - Sender email may not match authenticated SMTP account requirements
  - If `webContentLink` is not publicly accessible, recipients may not be able to download the PDF
  - Missing or malformed recipient email
  - Email may be flagged as spam depending on SMTP/domain setup
  - The email does not actually attach the PDF; it provides a link, despite the email copy stating the PDF is attached
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheets Trigger | googleSheetsTrigger | Poll Google Sheets for new registration rows |  | Make Copy of Template | 📊 **1. The Data Source**<br>This workflow starts when a new participant registers.<br>Ensure your data source (like Google Sheets or a Webhook) contains unique identifiers for the QR code (e.g., Ticket ID) and personal details (Name, Size, Blood Type).<br>🚀 **Automated e-Ticket PDF Generator > Created by Scale On Fajar**<br>This workflow automates the tedious process of generating event tickets. It takes incoming registration data, generates a dynamic QR Code, injects it into a Google Doc template along with personal details, and outputs a ready-to-send PDF file.<br>Perfect for: Marathons, webinars, workshops, or any event requiring personalized passes.<br>Tip: Work smarter, scale mindfully! |
| Make Copy of Template | googleDrive | Copy the Google Doc master template for the current registrant | Google Sheets Trigger | Change Custom Variables | **2. Duplicate & Inject Data**<br>1. We create a fresh copy of your master Google Doc template.<br>2. We replace text placeholders (like {{nama_lengkap}}) with real data.<br>3. We insert the generated QR Code binary image into the document.<br>Don't forget to map your own Google Doc Template ID here!<br>🚀 **Automated e-Ticket PDF Generator > Created by Scale On Fajar**<br>This workflow automates the tedious process of generating event tickets. It takes incoming registration data, generates a dynamic QR Code, injects it into a Google Doc template along with personal details, and outputs a ready-to-send PDF file.<br>Perfect for: Marathons, webinars, workshops, or any event requiring personalized passes.<br>Tip: Work smarter, scale mindfully! |
| Change Custom Variables | googleDocs | Replace placeholders inside the copied Google Doc | Make Copy of Template | Insert image | **2. Duplicate & Inject Data**<br>1. We create a fresh copy of your master Google Doc template.<br>2. We replace text placeholders (like {{nama_lengkap}}) with real data.<br>3. We insert the generated QR Code binary image into the document.<br>Don't forget to map your own Google Doc Template ID here!<br>🚀 **Automated e-Ticket PDF Generator > Created by Scale On Fajar**<br>This workflow automates the tedious process of generating event tickets. It takes incoming registration data, generates a dynamic QR Code, injects it into a Google Doc template along with personal details, and outputs a ready-to-send PDF file.<br>Perfect for: Marathons, webinars, workshops, or any event requiring personalized passes.<br>Tip: Work smarter, scale mindfully! |
| Insert image | httpRequest | Call Google Docs API to insert a QR code image into the document | Change Custom Variables | Generate PDF | **3. Dynamic QR Code**<br>This HTTP Request node calls an API to generate a QR Code image based on the participant's unique ID. The binary output is then passed to the document in the next steps.<br>🚀 **Automated e-Ticket PDF Generator > Created by Scale On Fajar**<br>This workflow automates the tedious process of generating event tickets. It takes incoming registration data, generates a dynamic QR Code, injects it into a Google Doc template along with personal details, and outputs a ready-to-send PDF file.<br>Perfect for: Marathons, webinars, workshops, or any event requiring personalized passes.<br>Tip: Work smarter, scale mindfully! |
| Generate PDF | googleDrive | Export the personalized Google Doc as PDF | Insert image | Add PDF To Drive | **🖨️4. Export, Save & Distribute**<br>Finally, the personalized Google Doc is downloaded as a PDF file and securely uploaded to a designated Google Drive folder. Once saved, the workflow uses the Mailketing node to automatically email the generated e-Ticket directly to the participant.<br>Pro-Tip: Make sure to connect your Mailketing/SMTP credentials and ensure your Google Drive folder is set to "Anyone with the link can view" so the attachment can be downloaded correctly.<br>🚀 **Automated e-Ticket PDF Generator > Created by Scale On Fajar**<br>This workflow automates the tedious process of generating event tickets. It takes incoming registration data, generates a dynamic QR Code, injects it into a Google Doc template along with personal details, and outputs a ready-to-send PDF file.<br>Perfect for: Marathons, webinars, workshops, or any event requiring personalized passes.<br>Tip: Work smarter, scale mindfully! |
| Add PDF To Drive | googleDrive | Upload the generated PDF into Google Drive | Generate PDF | Send an Email | **🖨️4. Export, Save & Distribute**<br>Finally, the personalized Google Doc is downloaded as a PDF file and securely uploaded to a designated Google Drive folder. Once saved, the workflow uses the Mailketing node to automatically email the generated e-Ticket directly to the participant.<br>Pro-Tip: Make sure to connect your Mailketing/SMTP credentials and ensure your Google Drive folder is set to "Anyone with the link can view" so the attachment can be downloaded correctly.<br>🚀 **Automated e-Ticket PDF Generator > Created by Scale On Fajar**<br>This workflow automates the tedious process of generating event tickets. It takes incoming registration data, generates a dynamic QR Code, injects it into a Google Doc template along with personal details, and outputs a ready-to-send PDF file.<br>Perfect for: Marathons, webinars, workshops, or any event requiring personalized passes.<br>Tip: Work smarter, scale mindfully! |
| Send an Email | emailSend | Email the participant a branded e-ticket message with download link | Add PDF To Drive |  | **🖨️4. Export, Save & Distribute**<br>Finally, the personalized Google Doc is downloaded as a PDF file and securely uploaded to a designated Google Drive folder. Once saved, the workflow uses the Mailketing node to automatically email the generated e-Ticket directly to the participant.<br>Pro-Tip: Make sure to connect your Mailketing/SMTP credentials and ensure your Google Drive folder is set to "Anyone with the link can view" so the attachment can be downloaded correctly.<br>🚀 **Automated e-Ticket PDF Generator > Created by Scale On Fajar**<br>This workflow automates the tedious process of generating event tickets. It takes incoming registration data, generates a dynamic QR Code, injects it into a Google Doc template along with personal details, and outputs a ready-to-send PDF file.<br>Perfect for: Marathons, webinars, workshops, or any event requiring personalized passes.<br>Tip: Work smarter, scale mindfully! |
| Sticky Note | stickyNote | Workspace annotation for overall workflow purpose |  |  | 🚀 **Automated e-Ticket PDF Generator > Created by Scale On Fajar**<br>This workflow automates the tedious process of generating event tickets. It takes incoming registration data, generates a dynamic QR Code, injects it into a Google Doc template along with personal details, and outputs a ready-to-send PDF file.<br>Perfect for: Marathons, webinars, workshops, or any event requiring personalized passes.<br>Tip: Work smarter, scale mindfully! |
| Sticky Note1 | stickyNote | Workspace annotation for input/data source guidance |  |  | 📊 **1. The Data Source**<br>This workflow starts when a new participant registers.<br>Ensure your data source (like Google Sheets or a Webhook) contains unique identifiers for the QR code (e.g., Ticket ID) and personal details (Name, Size, Blood Type). |
| Sticky Note2 | stickyNote | Workspace annotation for QR code insertion stage |  |  | **3. Dynamic QR Code**<br>This HTTP Request node calls an API to generate a QR Code image based on the participant's unique ID. The binary output is then passed to the document in the next steps. |
| Sticky Note3 | stickyNote | Workspace annotation for template duplication and variable injection |  |  | **2. Duplicate & Inject Data**<br>1. We create a fresh copy of your master Google Doc template.<br>2. We replace text placeholders (like {{nama_lengkap}}) with real data.<br>3. We insert the generated QR Code binary image into the document.<br>Don't forget to map your own Google Doc Template ID here! |
| Sticky Note4 | stickyNote | Workspace annotation for export, storage, and emailing |  |  | **🖨️4. Export, Save & Distribute**<br>Finally, the personalized Google Doc is downloaded as a PDF file and securely uploaded to a designated Google Drive folder. Once saved, the workflow uses the Mailketing node to automatically email the generated e-Ticket directly to the participant.<br>Pro-Tip: Make sure to connect your Mailketing/SMTP credentials and ensure your Google Drive folder is set to "Anyone with the link can view" so the attachment can be downloaded correctly. |

---

## 4. Reproducing the Workflow from Scratch

1. **Prepare the source Google Sheet**
   1. Create a Google Sheet to store event registrations.
   2. Add a worksheet/tab for incoming records, for example `QR Code Data`.
   3. Ensure the sheet contains at least these columns:
      - `TIKET`
      - `NAMA`
      - `Email`
   4. If your ticket template also needs more participant fields, add those columns too.

2. **Prepare the Google Docs template**
   1. Create a Google Doc that will act as the master e-ticket template.
   2. Add placeholder text where dynamic content should appear.
   3. In this workflow, at minimum include the placeholder:
      - `{kode_qr}`
   4. If you want more fields such as name or event title, add placeholders and extend the Google Docs replacement node accordingly.
   5. Note the template file ID from the document URL.

3. **Prepare the destination Google Drive folder**
   1. Create a folder in Google Drive, for example `eTicket`.
   2. Note its folder ID.
   3. If recipients must download from a link, set sharing appropriately, such as “Anyone with the link can view”, or use another controlled sharing approach.

4. **Create the trigger node**
   1. Add a **Google Sheets Trigger** node.
   2. Connect Google Sheets Trigger OAuth2 credentials.
   3. Select the spreadsheet containing registrations.
   4. Select the worksheet/tab to monitor.
   5. Set polling frequency to **every minute**.
   6. Test that a new row produces fields like `TIKET`, `NAMA`, and `Email`.

5. **Create the template copy node**
   1. Add a **Google Drive** node named `Make Copy of Template`.
   2. Set operation to **Copy**.
   3. Select the Google Docs template file.
   4. Set the new file name to:
      `eTicket for {{ $json.TIKET }}`
   5. Connect Google Drive OAuth2 credentials.
   6. Connect `Google Sheets Trigger` → `Make Copy of Template`.

6. **Create the placeholder replacement node**
   1. Add a **Google Docs** node named `Change Custom Variables`.
   2. Set operation to **Update**.
   3. Set the target document to the copied file using:
      `={{ $json.id }}`
   4. Add a replace action:
      - Find: `{kode_qr}`
      - Replace with: `={{ $('Google Sheets Trigger').item.json.TIKET }}`
   5. Connect Google Docs OAuth2 credentials.
   6. Connect `Make Copy of Template` → `Change Custom Variables`.

7. **Create the QR image insertion node**
   1. Add an **HTTP Request** node named `Insert image`.
   2. Set method to **POST**.
   3. Set URL to:
      `=https://docs.googleapis.com/v1/documents/{{$json.documentId}}:batchUpdate`
   4. Enable authentication using **Predefined Credential Type**.
   5. Choose credential type **Google Docs OAuth2 API**.
   6. Enable body sending.
   7. Add body parameter:
      - Name: `requests`
      - Value:
        `={{ [{ "insertInlineImage": { "uri": "https://quickchart.io/qr?format=png&text=$('Google Sheets Trigger').item.json.TIKET", "location": { "segmentId": "", "index": 27 } } }] }}`
   8. Set response handling to return full response as text if you want to mirror this workflow exactly.
   9. Connect `Change Custom Variables` → `Insert image`.

8. **Important template alignment check**
   1. The QR image is inserted at document index `27`.
   2. This index is template-dependent.
   3. If the QR code appears in the wrong location or insertion fails, adjust the index.
   4. A safer alternative is to build a more advanced Docs API sequence that replaces a known placeholder region instead of relying on a fixed index.

9. **Create the PDF export node**
   1. Add a **Google Drive** node named `Generate PDF`.
   2. Set operation to **Download**.
   3. Set file ID to:
      `={{ $('Change Custom Variables').item.json.documentId }}`
   4. Under Google file conversion, enable export from Google Docs to:
      - `application/pdf`
   5. Set binary property name to `data`.
   6. Connect Google Drive OAuth2 credentials.
   7. Connect `Insert image` → `Generate PDF`.

10. **Create the PDF upload node**
    1. Add another **Google Drive** node named `Add PDF To Drive`.
    2. Configure it to upload the incoming binary PDF to your chosen Drive folder.
    3. Select:
       - Drive: `My Drive`
       - Folder: your `eTicket` folder
    4. Set the file name to:
       `={{ $('Make Copy of Template').item.json.name }}`
    5. Ensure the node uses the binary data generated by the previous node, typically the `data` property.
    6. Connect `Generate PDF` → `Add PDF To Drive`.

11. **Verify uploaded file metadata**
    1. Test the upload node.
    2. Confirm its output includes a usable file link such as `webContentLink`.
    3. If the link is absent, adjust the node or Drive sharing settings accordingly.

12. **Create the email node**
    1. Add an **Email Send** node named `Send an Email`.
    2. Connect SMTP credentials.
    3. Configure:
       - From: your sender email, for example `user@example.com`
       - To:
         `={{ $('Google Sheets Trigger').item.json.Email }}`
       - Subject:
         `e-Ticket Resmi Anda untuk Event [Nama Event] 🎟️`
    4. Paste the HTML email body.
    5. Ensure it references:
       - Recipient name: `{{ $('Google Sheets Trigger').item.json.NAMA }}`
       - Ticket ID: `{{ $('Google Sheets Trigger').item.json.TIKET }}`
       - Download link: `{{ $json.webContentLink }}`
    6. Connect `Add PDF To Drive` → `Send an Email`.

13. **Check SMTP compatibility**
    1. Make sure the authenticated SMTP account is allowed to send using the configured `fromEmail`.
    2. If not, use the exact sender identity permitted by your SMTP provider.

14. **Activate and test the full chain**
    1. Add a new row to the monitored Google Sheet.
    2. Wait for the poll cycle.
    3. Verify the workflow:
       - Copies the template
       - Replaces placeholder text
       - Inserts the QR image
       - Exports to PDF
       - Uploads the PDF
       - Sends the email

15. **Recommended improvements for production**
    1. Add an **IF** node or validation step before document generation to ensure `Email` and `TIKET` are not empty.
    2. Add error handling branches or an error workflow.
    3. Update the email wording if you do not actually attach the PDF.
    4. Add more placeholder replacements for participant data beyond `{kode_qr}`.
    5. Consider de-duplication logic so the same row is not processed twice.
    6. Optionally add a delay between document update and PDF export if Google Docs propagation causes missing images in exported PDFs.

### Required Credentials
- **Google Sheets Trigger OAuth2**  
  For monitoring new sheet rows.
- **Google Drive OAuth2**  
  For copying templates, exporting docs, and uploading PDFs.
- **Google Docs OAuth2**  
  For placeholder replacement and Docs API image insertion.
- **SMTP**  
  For sending emails.

### Input/Output Expectations
- **Input from sheet:** one item per registration row with at least `TIKET`, `NAMA`, `Email`
- **Intermediate outputs:**
  - Template copy node returns new file metadata including file ID
  - Google Docs update node returns document metadata including `documentId`
  - PDF export node returns binary `data`
  - Drive upload node returns uploaded file metadata including shareable link fields
- **Final output:** one email sent to the participant with a PDF download link

### Sub-workflow Setup
This workflow does **not** use any Execute Workflow or other sub-workflow nodes. No separate sub-workflow configuration is required.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated e-Ticket PDF Generator created by Scale On Fajar | Workspace branding note |
| Intended for marathons, webinars, workshops, or any event requiring personalized passes | Use-case guidance |
| Suggested source systems include Google Sheets or a Webhook, with unique ticket identifiers and personal details | Input design note |
| Template mapping reminder: update the Google Doc template ID and placeholder fields to match your own document | Template configuration note |
| QR code image source is generated through QuickChart | https://quickchart.io/ |
| Distribution note: configure your Google Drive sharing so recipients can access the generated file link | Google Drive sharing setup |
| The email says the PDF is attached, but the implemented flow sends a download link rather than an actual file attachment | Important behavior clarification |