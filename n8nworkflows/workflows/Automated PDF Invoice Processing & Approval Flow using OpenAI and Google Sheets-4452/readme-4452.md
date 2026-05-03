Automated PDF Invoice Processing & Approval Flow using OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/automated-pdf-invoice-processing---approval-flow-using-openai-and-google-sheets-4452


# Automated PDF Invoice Processing & Approval Flow using OpenAI and Google Sheets

---

### 1. Workflow Overview

This workflow automates the end-to-end processing and approval of PDF invoices using AI and Google services. Its primary use case targets businesses seeking to streamline invoice receipt, data extraction, approval workflows, and recordkeeping with minimal manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Invoice Collection**: Monitors multiple sources (Google Drive folder, Gmail inbox, and web form) for incoming PDF invoices.
- **1.2 File Handling & Text Extraction**: Downloads or accesses PDF files and extracts raw text content from them.
- **1.3 AI Data Parsing**: Utilizes OpenAI GPT-4o-mini via LangChain nodes to parse invoice text into structured JSON data including invoice details and category classification.
- **1.4 Approval Request & Decision**: Sends an approval email with an embedded form to a reviewer; captures decision and notes.
- **1.5 Decision Routing and Notification**: Routes invoice data for storage if approved; sends rejection alerts if declined.
- **1.6 Data Storage**: Appends parsed and approved invoice data into a structured Google Sheets document for recordkeeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Invoice Collection

- **Overview**: This block triggers the workflow by detecting new invoices arriving via three channels: a specific Google Drive folder, Gmail with attachments, and a web form upload.
- **Nodes Involved**:
  - Invoice Folder Monitor (Google Drive Trigger)
  - Monitor Email Attachments (Gmail Trigger)
  - Upload Invoice (PDF) Form (Form Trigger)
  - Download Invoice PDF (Google Drive node)
  
- **Node Details**:

  - **Invoice Folder Monitor**  
    - Type: Google Drive Trigger  
    - Role: Watches a specific Google Drive folder for newly added PDF files.  
    - Configuration: Polls every minute; watches folder ID `1KJ4fvXcKVMGJunsKvPYf8PkX5K9SVwFk`.  
    - Inputs: None (trigger)  
    - Outputs: Triggers downstream nodes with file metadata.  
    - Edge Cases: Folder ID changes or permission issues may cause trigger failure.
  
  - **Monitor Email Attachments**  
    - Type: Gmail Trigger  
    - Role: Monitors Gmail inbox for new emails with PDF attachments.  
    - Configuration: Polls every minute; downloads attachment(s) automatically with prefix `attachment_`.  
    - Inputs: None (trigger)  
    - Outputs: Email data including binary attachments.  
    - Edge Cases: Gmail API quota limits, attachment download errors, non-PDF attachments.
  
  - **Upload Invoice (PDF) Form**  
    - Type: Form Trigger  
    - Role: Provides a web form for users to upload invoice PDFs manually.  
    - Configuration: Single PDF file upload required.  
    - Inputs: None (trigger)  
    - Outputs: Binary PDF data from form submission.  
    - Edge Cases: Upload failures, unsupported file types, multiple file uploads disallowed.
  
  - **Download Invoice PDF**  
    - Type: Google Drive node  
    - Role: Downloads the invoice PDF file from Google Drive using file ID.  
    - Configuration: File ID taken dynamically from trigger data.  
    - Inputs: File metadata from Invoice Folder Monitor  
    - Outputs: Binary PDF file for processing.  
    - Edge Cases: File permissions, deleted or moved files, download timeouts.

#### 2.2 File Handling & Text Extraction

- **Overview**: Extracts raw text from PDFs obtained via Drive, Email, or Form to prepare for AI parsing.
- **Nodes Involved**:
  - Extract Text from Drive PDF
  - Extract Text from Email PDF
  - Extract Text from Form PDF

- **Node Details**:

  - **Extract Text from Drive PDF**  
    - Type: Extract From File  
    - Role: Extracts text from binary PDF downloaded from Drive.  
    - Configuration: PDF operation, default options.  
    - Inputs: Binary PDF from Download Invoice PDF  
    - Outputs: JSON containing extracted raw text.  
    - Edge Cases: Corrupted PDFs, extraction failures.
  
  - **Extract Text from Email PDF**  
    - Type: Extract From File  
    - Role: Extracts text from PDF attachment binary from Gmail.  
    - Configuration: PDF operation, binary property set to `attachment_0` (first attachment).  
    - Inputs: Binary PDF from Monitor Email Attachments  
    - Outputs: JSON with raw extracted text.  
    - Edge Cases: Multiple attachments, non-PDF attachments, extraction errors.
  
  - **Extract Text from Form PDF**  
    - Type: Extract From File  
    - Role: Extracts text from PDF uploaded via the form.  
    - Configuration: PDF operation, binary property set to `Upload` (form file upload).  
    - Inputs: Binary PDF from Upload Invoice (PDF) Form  
    - Outputs: JSON with raw extracted text.  
    - Edge Cases: Upload errors, corrupted files.

#### 2.3 AI Data Parsing

- **Overview**: Parses the extracted raw text using OpenAI GPT (GPT-4o-mini) with LangChain agent to extract invoice fields and categorize the invoice.
- **Nodes Involved**:
  - OpenAI Chat Model
  - Structured Output Parser
  - Invoice Parser AI Agent

- **Node Details**:

  - **OpenAI Chat Model**  
    - Type: LangChain Language Model (OpenAI GPT)  
    - Role: Provides GPT-4o-mini model for natural language understanding.  
    - Configuration: Model set to `gpt-4o-mini`; uses OpenAI API credentials.  
    - Inputs: Prompts from LangChain Agent  
    - Outputs: AI-generated text responses.  
    - Edge Cases: API rate limits, invalid API keys, network issues.
  
  - **Structured Output Parser**  
    - Type: LangChain Output Parser (Structured JSON)  
    - Role: Enforces output to match JSON schema for invoice fields.  
    - Configuration: JSON schema example with invoice_number, dates, vendor_name, items, tax, category, etc.  
    - Inputs: AI-generated text from OpenAI Chat Model  
    - Outputs: Parsed JSON matching schema.  
    - Edge Cases: Parsing errors if AI output deviates from schema.
  
  - **Invoice Parser AI Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates prompt with extracted text, calls OpenAI Chat Model with instructions to parse invoice data.  
    - Configuration: Custom prompt instructing to extract fields and return JSON in strict format. Output parsing enabled.  
    - Inputs: Raw text from PDF extraction nodes  
    - Outputs: Structured JSON invoice data  
    - Edge Cases: Misinterpretation of invoice text, incomplete data, AI timeouts.

#### 2.4 Approval Request & Decision

- **Overview**: Sends an email approval request with a form to a designated approver, capturing their approval decision, reviewer name, and notes.
- **Nodes Involved**:
  - Send Invoice for Approval
  - Check Approval Decision

- **Node Details**:

  - **Send Invoice for Approval**  
    - Type: Gmail node (sendAndWait operation)  
    - Role: Sends an email with embedded approval form (dropdown Yes/No, reviewer name, notes). Waits for response.  
    - Configuration:  
      - Recipient email is placeholder `replace_with_approver_email@yopmail.com`.  
      - Subject dynamically includes vendor name.  
      - Message prompts review.  
      - Form fields: Approve Invoice? (dropdown, required), Reviewed By (required), Approval Notes (textarea).  
    - Inputs: Parsed invoice JSON from AI Agent  
    - Outputs: Approval form response data.  
    - Edge Cases: Email delivery failure, form response timeout, invalid input by user.
  
  - **Check Approval Decision**  
    - Type: If node  
    - Role: Evaluates approval response; routes workflow accordingly.  
    - Configuration: Checks if `Approve Invoice?` equals "Yes".  
    - Inputs: Approval form data from Send Invoice for Approval  
    - Outputs: Two branches: approved (Yes) and rejected (No or other).  
    - Edge Cases: Missing or malformed approval data.

#### 2.5 Decision Routing and Notification

- **Overview**: Based on approval, either saves data or notifies finance team of rejection.
- **Nodes Involved**:
  - Insert Invoice Data
  - Send Rejection Alert

- **Node Details**:

  - **Insert Invoice Data**  
    - Type: Google Sheets node  
    - Role: Appends all invoice data plus approval info to a Google Sheets document.  
    - Configuration:  
      - Target spreadsheet ID `1ueJfN5dFTXY3_AdvnYUL5_RjV9YwSFvbxwA_ivtqnJk`  
      - Sheet name "Invoices" (gid=0)  
      - Columns mapped from AI output and approval form response, including invoice fields, category, approval status, notes, reviewer.  
    - Inputs: Approved invoice data and approval form data  
    - Outputs: Confirmation of append operation  
    - Edge Cases: Sheet permission issues, invalid data types, quota limits.
  
  - **Send Rejection Alert**  
    - Type: Gmail node  
    - Role: Sends notification to finance team upon invoice rejection.  
    - Configuration:  
      - Sends to `finance_team@yopmail.com`  
      - Subject includes vendor name  
      - Message includes reviewer name and approval notes for context.  
      - Plain text email type.  
    - Inputs: Rejected invoice data and approval form data  
    - Outputs: Email sent confirmation  
    - Edge Cases: Email delivery failure.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                                  | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                                   |
|---------------------------|----------------------------------|-------------------------------------------------|----------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Invoice Folder Monitor     | Google Drive Trigger             | Trigger on new PDFs in Drive folder              | None                             | Download Invoice PDF                | Setup: Google Drive Credential required                                                                       |
| Download Invoice PDF       | Google Drive                    | Download PDF file from Drive                      | Invoice Folder Monitor           | Extract Text from Drive PDF         |                                                                                                               |
| Extract Text from Drive PDF| Extract From File                | Extract text from Drive PDF binary                | Download Invoice PDF             | Invoice Parser AI Agent             |                                                                                                               |
| Monitor Email Attachments  | Gmail Trigger                   | Trigger on Gmail emails with PDF attachments      | None                             | Extract Text from Email PDF         | Setup: Gmail Credential required                                                                               |
| Extract Text from Email PDF| Extract From File                | Extract text from email PDF attachment             | Monitor Email Attachments        | Invoice Parser AI Agent             |                                                                                                               |
| Upload Invoice (PDF) Form  | Form Trigger                   | Web form for manual PDF upload                     | None                             | Extract Text from Form PDF          |                                                                                                               |
| Extract Text from Form PDF | Extract From File               | Extract text from form-uploaded PDF                 | Upload Invoice (PDF) Form        | Invoice Parser AI Agent             |                                                                                                               |
| OpenAI Chat Model          | LangChain Language Model (OpenAI)| Provides GPT-4o-mini for invoice parsing           | Invoice Parser AI Agent (prompt) | Invoice Parser AI Agent (response) | Setup: OpenAI API Key (GPT-4) required                                                                        |
| Structured Output Parser   | LangChain Output Parser         | Enforces JSON schema on AI output                   | OpenAI Chat Model               | Invoice Parser AI Agent             |                                                                                                               |
| Invoice Parser AI Agent    | LangChain Agent                 | Parses raw invoice text into structured data       | Extract Text from * PDF nodes    | Send Invoice for Approval           |                                                                                                               |
| Send Invoice for Approval  | Gmail (sendAndWait)             | Sends approval request email with form             | Invoice Parser AI Agent          | Check Approval Decision             | Setup: Gmail Credential required                                                                               |
| Check Approval Decision    | If                             | Routes flow based on approval Yes/No                | Send Invoice for Approval        | Insert Invoice Data / Send Rejection Alert |                                                                                                               |
| Insert Invoice Data        | Google Sheets                   | Appends invoice + approval data to Google Sheet    | Check Approval Decision (Yes)    | None                              | Setup: Google Sheets Credential required                                                                      |
| Send Rejection Alert       | Gmail                          | Sends rejection notification to finance team       | Check Approval Decision (No)     | Insert Invoice Data                 |                                                                                                               |
| Sticky Note2               | Sticky Note                    | Setup instructions for Google Sheets and credentials| None                             | None                              | Google Sheets Structure and Required Credentials                                                             |
| Sticky Note3               | Sticky Note                    | Workflow purpose and detailed description           | None                             | None                              | Overview of workflow capabilities and AI usage                                                                |
| Sticky Note4               | Sticky Note                    | Detailed workflow process overview                   | None                             | None                              | Stepwise explanation of workflow blocks                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Drive Trigger node ("Invoice Folder Monitor")**  
   - Type: Google Drive Trigger  
   - Configure to trigger on file creation in folder with ID `1KJ4fvXcKVMGJunsKvPYf8PkX5K9SVwFk`  
   - Poll every minute  
   - Credential: Google Drive OAuth2  

2. **Create a Google Drive node ("Download Invoice PDF")**  
   - Operation: Download file  
   - File ID: Use expression `{{$json["id"]}}` from trigger  
   - Credential: Google Drive OAuth2  
   - Connect input from "Invoice Folder Monitor"  

3. **Create Extract From File nodes for each input source:**

   a. "Extract Text from Drive PDF"  
   - Operation: pdf extraction  
   - Input: Binary file from "Download Invoice PDF"  
   
   b. "Extract Text from Email PDF"  
   - Operation: pdf extraction  
   - Binary property: `attachment_0`  
   - Input from Gmail Trigger (created later)  
   
   c. "Extract Text from Form PDF"  
   - Operation: pdf extraction  
   - Binary property: `Upload`  
   - Input from Form Trigger (created later)  

4. **Create Gmail Trigger node ("Monitor Email Attachments")**  
   - Configure to poll Gmail inbox every minute  
   - Enable download attachments with prefix `attachment_`  
   - Credential: Gmail OAuth2  

5. **Create Form Trigger node ("Upload Invoice (PDF) Form")**  
   - Create a form with a single required file field accepting `.pdf`  
   - Credential: none needed  
   - Position to trigger extraction node  

6. **Create LangChain Language Model node ("OpenAI Chat Model")**  
   - Model: GPT-4o-mini  
   - Credential: OpenAI API Key (GPT-4)  

7. **Create LangChain Structured Output Parser node ("Structured Output Parser")**  
   - Provide sample JSON schema with invoice fields (invoice_number, invoice_date, due_date, vendor_name, total_amount, currency, items array, tax, category)  

8. **Create LangChain Agent node ("Invoice Parser AI Agent")**  
   - Prompt: Instructions to parse raw invoice text into structured JSON matching schema  
   - Enable output parser, linking to "Structured Output Parser"  
   - Connect inputs from all three extraction nodes (Drive, Email, Form)  
   - Connect AI model and output parser to agent  

9. **Create Gmail node ("Send Invoice for Approval")**  
   - Operation: sendAndWait (waits for form response)  
   - Recipient email: set to approver's email  
   - Subject: include vendor name dynamically  
   - Message: brief text indicating action needed  
   - Form fields:  
     - Dropdown "Approve Invoice?" options Yes/No, required  
     - Text "Reviewed By", required  
     - Textarea "Approval Notes", optional  
   - Credential: Gmail OAuth2  
   - Connect input from "Invoice Parser AI Agent"  

10. **Create If node ("Check Approval Decision")**  
    - Condition: Check if `Approve Invoice?` == "Yes"  
    - Connect input from "Send Invoice for Approval"  

11. **Create Google Sheets node ("Insert Invoice Data")**  
    - Operation: Append  
    - Spreadsheet ID: `1ueJfN5dFTXY3_AdvnYUL5_RjV9YwSFvbxwA_ivtqnJk`  
    - Sheet Name: "Invoices" (gid=0)  
    - Map columns from AI output and approval form fields:  
      Invoice Number, Invoice Date, Due Date, Vendor Name, Total Amount, Currency, Items, Tax, Category, Approved, Approval Notes, Reviewed By  
    - Credential: Google Sheets OAuth2  
    - Connect input from "Check Approval Decision" (Yes branch)  

12. **Create Gmail node ("Send Rejection Alert")**  
    - Recipient: `finance_team@yopmail.com`  
    - Subject: Alert with vendor name  
    - Message: Includes reviewer name and approval notes  
    - Plain text email  
    - Credential: Gmail OAuth2  
    - Connect input from "Check Approval Decision" (No branch)  

13. **Connect "Send Rejection Alert" output to "Insert Invoice Data"**  
    - To append rejected invoice data as well into Google Sheets  

14. **Add Sticky Notes** for setup instructions and workflow overview as per convenience.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Google Sheets Structure: Sheet named "Invoices" with columns A-L for invoice fields and approval data. Required credentials: Google Drive, Gmail, Google Sheets, OpenAI API key (GPT-4).                                                                                                                                                                                                             | Setup instructions from Sticky Note2                                                                        |
| This workflow monitors Google Drive, Gmail, and a web form for incoming invoices, extracts key data using GPT-4o-mini, sends approval emails with forms, and stores results into Google Sheets. It provides a comprehensive automated invoice processing and approval pipeline suitable for SMBs and enterprises.                                                                                  | Sticky Note3 detailed workflow capabilities                                                                 |
| Workflow process overview includes invoice collection, file handling, text extraction, AI parsing, approval requests, decision routing, and data storage, with polling every minute for responsiveness.                                                                                                                                                                                               | Sticky Note4                                                                                                 |
| AI prompt instructs GPT to output invoice data in strict JSON format including invoice number, dates, vendor, amounts, items, tax, and categorization into types like Utilities, Travel, Office Supplies, Food & Beverage, Others. This structure ensures consistent parsing and downstream processing.                                                                                                  | LangChain Agent prompt in "Invoice Parser AI Agent" node                                                    |
| Replace placeholder email addresses (approver and finance team) with real addresses before deployment.                                                                                                                                                                                                                                                                                            | "Send Invoice for Approval" and "Send Rejection Alert" node parameters                                      |
| Ensure all OAuth2 credentials (Google Drive, Gmail, Google Sheets) are correctly configured and authorized with necessary API scopes to avoid permission errors.                                                                                                                                                                                                                                  | Credential setup notes                                                                                        |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---