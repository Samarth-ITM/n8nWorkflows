Translate PDFs into Multiple Languages with Google Translate & ConvertAPI Cost Tracking

https://n8nworkflows.xyz/workflows/translate-pdfs-into-multiple-languages-with-google-translate---convertapi-cost-tracking-10864


# Translate PDFs into Multiple Languages with Google Translate & ConvertAPI Cost Tracking

### 1. Workflow Overview

This workflow automates the translation of an English PDF document into two different target languages using Google Translate, then converts the translated HTML content back into PDFs via ConvertAPI, and finally emails the translated documents with cost tracking. It is designed for use cases where users submit a form with a PDF and language choices, and require translated PDFs delivered by email, along with a detailed record of translation costs.

Logical blocks of the workflow:

- **1.1 Input Reception and Processing**: Receiving PDF and form data, extracting text from the PDF.
- **1.2 Content Conversion**: Converting extracted text to well-structured HTML.
- **1.3 Translation**: Translating the HTML content into two specified languages via Google Translate.
- **1.4 PDF Generation**: Converting translated HTML into PDFs using ConvertAPI.
- **1.5 Document Preparation and Merging**: Finalizing PDFs and optionally merging documents.
- **1.6 Email Preparation and Sending**: Preparing email content and sending translated PDFs to recipients.
- **1.7 Cost Tracking**: Calculating and recording the cost of translation and PDF conversion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Processing

- **Overview:**  
  This block begins the workflow by receiving a PDF and language choices from a form trigger, processing the form data, and extracting raw text from the PDF file for translation.

- **Nodes Involved:**  
  - Document Translation Request (Form Trigger)  
  - Process Form Data  
  - Extract Text from PDF

- **Node Details:**

  - **Document Translation Request (Form Trigger)**  
    - *Type:* Form Trigger  
    - *Role:* Entry point triggered by form submission containing the PDF and language selection.  
    - *Configuration:* Webhook-based trigger with a unique webhook ID.  
    - *Inputs:* Form submission data including PDF file and language parameters.  
    - *Outputs:* Passes form data to "Process Form Data".  
    - *Failure types:* Webhook errors, missing file or parameters.  

  - **Process Form Data**  
    - *Type:* Code  
    - *Role:* Processes and formats the incoming form data for downstream nodes.  
    - *Configuration:* Custom JavaScript code to parse form inputs and prepare data structure.  
    - *Inputs:* Raw form submission from trigger.  
    - *Outputs:* Passes processed data to "Extract Text from PDF".  
    - *Failure types:* Expression errors, unexpected form data structure.  

  - **Extract Text from PDF**  
    - *Type:* Extract From File  
    - *Role:* Extracts raw textual content from the uploaded PDF file.  
    - *Configuration:* Default extraction settings, targets text extraction.  
    - *Inputs:* Processed PDF file data from "Process Form Data".  
    - *Outputs:* Raw text content to "Convert to HTML".  
    - *Failure types:* PDF corrupt or encrypted, unsupported PDF format.

#### 1.2 Content Conversion

- **Overview:**  
  Converts the extracted PDF text into properly formatted HTML with CSS styling to preserve layout for translation and PDF reconversion.

- **Nodes Involved:**  
  - Convert to HTML

- **Node Details:**

  - **Convert to HTML**  
    - *Type:* Code  
    - *Role:* Converts plain text extracted from PDF into styled HTML suitable for translation and PDF regeneration.  
    - *Configuration:* JavaScript code generating HTML structure with CSS for styling.  
    - *Inputs:* Extracted text from "Extract Text from PDF".  
    - *Outputs:* HTML content to "Translate to Language 1" and "Has 2nd Language?" conditional node.  
    - *Failure types:* Code errors, malformed HTML generation.

#### 1.3 Translation

- **Overview:**  
  Translates the HTML content to one or two target languages using Google Translate API, preserving HTML tags and structure.

- **Nodes Involved:**  
  - Translate to Language 1  
  - Has 2nd Language? (Conditional)  
  - Translate to Language 2

- **Node Details:**

  - **Translate to Language 1**  
    - *Type:* Google Translate  
    - *Role:* Translates HTML content into the first target language.  
    - *Configuration:* Uses Google Service Account credentials with Translation API enabled. Input is HTML preserving tags.  
    - *Inputs:* HTML from "Convert to HTML".  
    - *Outputs:* Translated HTML to "Prepare HTML 1".  
    - *Failure types:* Authentication errors, quota limits, malformed input.  

  - **Has 2nd Language?**  
    - *Type:* If  
    - *Role:* Checks if a second target language is specified to branch workflow accordingly.  
    - *Configuration:* Conditional expression based on form data language 2 presence.  
    - *Inputs:* Post first language translation step.  
    - *Outputs:* If true, proceeds to "Translate to Language 2", else skips.  
    - *Failure types:* Expression errors if input data incomplete.  

  - **Translate to Language 2**  
    - *Type:* Google Translate  
    - *Role:* Translates HTML content into the second target language when required.  
    - *Configuration:* Same as Language 1 translation node.  
    - *Inputs:* Original HTML from "Convert to HTML".  
    - *Outputs:* Translated HTML to "Prepare HTML 2".  
    - *Failure types:* Same as Language 1.

#### 1.4 PDF Generation

- **Overview:**  
  Converts the translated HTML content into PDF documents using the ConvertAPI service, preparing the HTML as binary data for conversion.

- **Nodes Involved:**  
  - Prepare HTML 1  
  - Prepare HTML 2  
  - Convert HTML to PDF 1  
  - Convert HTML to PDF 2  
  - Finalize PDF 1  
  - Finalize PDF 2

- **Node Details:**

  - **Prepare HTML 1 & Prepare HTML 2**  
    - *Type:* Code  
    - *Role:* Converts translated HTML content into binary format required by ConvertAPI for PDF creation.  
    - *Configuration:* JavaScript code wrapping HTML in binary data structure.  
    - *Inputs:* Translated HTML from respective translation nodes.  
    - *Outputs:* Binary HTML to respective ConvertAPI request nodes.  
    - *Failure types:* Data conversion errors.  

  - **Convert HTML to PDF 1 & Convert HTML to PDF 2**  
    - *Type:* HTTP Request  
    - *Role:* Sends prepared HTML to ConvertAPI HTTP endpoint to generate PDF with proper margins.  
    - *Configuration:* HTTP POST with ConvertAPI secret passed as query parameter or credential, specifying PDF conversion parameters.  
    - *Inputs:* Binary HTML from "Prepare HTML" nodes.  
    - *Outputs:* PDF binary data to respective "Finalize PDF" nodes.  
    - *Failure types:* Authentication failures, API quota limits, network timeouts.  

  - **Finalize PDF 1 & Finalize PDF 2**  
    - *Type:* Code  
    - *Role:* Finalizes the converted PDF data, possibly setting correct metadata or formatting for email attachment.  
    - *Configuration:* JavaScript code to set correct mime types and prepare output.  
    - *Inputs:* PDF binary from ConvertAPI HTTP request nodes.  
    - *Outputs:* PDF files merged in next step or sent via email.  
    - *Failure types:* Data corruption, incorrect mime type handling.

#### 1.5 Document Preparation and Merging

- **Overview:**  
  Merges the PDFs if two languages are translated and prepares the email content for sending.

- **Nodes Involved:**  
  - Merge Documents  
  - Prepare Email

- **Node Details:**

  - **Merge Documents**  
    - *Type:* Merge  
    - *Role:* Combines the two finalized PDF files into one data stream if both languages are translated, or passes single PDF forward.  
    - *Configuration:* Default merge settings to combine PDF binaries.  
    - *Inputs:* Finalized PDFs 1 and 2.  
    - *Outputs:* Merged PDFs to "Prepare Email".  
    - *Failure types:* Merge failures if input PDFs malformed or missing.  

  - **Prepare Email**  
    - *Type:* Code  
    - *Role:* Constructs email subject, body, and attachments data including merged PDFs and cost records.  
    - *Configuration:* JavaScript code formatting email content dynamically.  
    - *Inputs:* Merged PDFs and cost record from "Cost – Build record".  
    - *Outputs:* Passes email data to cost tracking and send email nodes.  
    - *Failure types:* Expression errors, incomplete data.

#### 1.6 Email Preparation and Sending

- **Overview:**  
  Determines whether to send one or two attachments and sends the email via Gmail using OAuth2 credentials.

- **Nodes Involved:**  
  - Cost – Build record  
  - Data Table  
  - Send Two Attachments? (If)  
  - Send Email (2 PDFs)  
  - Send Email (1 PDF)

- **Node Details:**

  - **Cost – Build record**  
    - *Type:* Code  
    - *Role:* Calculates and formats cost data for translation and PDF conversion for record-keeping and reporting.  
    - *Configuration:* JavaScript code computing costs based on API usage and document size.  
    - *Inputs:* Email preparation output.  
    - *Outputs:* Cost data to "Data Table" and "Send Two Attachments?" nodes.  
    - *Failure types:* Calculation errors, missing input data.  

  - **Data Table**  
    - *Type:* Data Table  
    - *Role:* Displays or logs the cost and usage data in tabular form.  
    - *Inputs:* Cost data from "Cost – Build record".  
    - *Outputs:* None further in workflow.  

  - **Send Two Attachments?**  
    - *Type:* If  
    - *Role:* Determines if the email should include two PDFs or one based on presence of second translation.  
    - *Inputs:* Cost data and email data.  
    - *Outputs:* Routes to either "Send Email (2 PDFs)" or "Send Email (1 PDF)".  
    - *Failure types:* Expression errors.  

  - **Send Email (2 PDFs) & Send Email (1 PDF)**  
    - *Type:* Gmail  
    - *Role:* Sends the email with one or two PDF attachments via Gmail SMTP with OAuth2 authentication.  
    - *Configuration:* Requires Gmail OAuth2 credentials connected in n8n.  
    - *Inputs:* Email content and attachments.  
    - *Outputs:* Confirmation of email sent (webhook response).  
    - *Failure types:* Authentication failures, SMTP errors, attachment size limits.

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                                 | Input Node(s)                         | Output Node(s)                               | Sticky Note                                                                                   |
|-----------------------------------|-----------------------|------------------------------------------------|-------------------------------------|----------------------------------------------|-----------------------------------------------------------------------------------------------|
| Document Translation Request (Form Trigger) | Form Trigger          | Entry point receiving PDF and language choices | None                                | Process Form Data                            |                                                                                               |
| Process Form Data                  | Code                  | Processes form data                             | Document Translation Request         | Extract Text from PDF                         |                                                                                               |
| Extract Text from PDF              | Extract From File     | Extracts text from PDF                          | Process Form Data                   | Convert to HTML                              |                                                                                               |
| Convert to HTML                   | Code                  | Converts text to formatted HTML                 | Extract Text from PDF               | Translate to Language 1, Has 2nd Language?  | Converts extracted PDF text to properly formatted HTML with CSS styling                        |
| Translate to Language 1           | Google Translate       | Translates HTML to first target language        | Convert to HTML                    | Prepare HTML 1                              | Translates HTML content preserving all tags and structure; requires Google Service Account     |
| Has 2nd Language?                 | If                     | Checks if second language specified              | Convert to HTML                    | Translate to Language 2 (if true)            |                                                                                               |
| Translate to Language 2           | Google Translate       | Translates HTML to second target language       | Has 2nd Language?                  | Prepare HTML 2                              | Translates HTML content preserving all tags and structure; requires Google Service Account     |
| Prepare HTML 1                   | Code                  | Prepares HTML binary for PDF conversion          | Translate to Language 1            | Convert HTML to PDF 1                        | Prepares HTML as binary for PDF conversion                                                    |
| Prepare HTML 2                   | Code                  | Prepares HTML binary for PDF conversion          | Translate to Language 2            | Convert HTML to PDF 2                        | Prepares HTML as binary for PDF conversion                                                    |
| Convert HTML to PDF 1             | HTTP Request           | Converts HTML to PDF via ConvertAPI              | Prepare HTML 1                    | Finalize PDF 1                              | Converts HTML to PDF using ConvertAPI with proper margins; requires ConvertAPI Secret          |
| Convert HTML to PDF 2             | HTTP Request           | Converts HTML to PDF via ConvertAPI              | Prepare HTML 2                    | Finalize PDF 2                              | Converts HTML to PDF using ConvertAPI with proper margins; requires ConvertAPI Secret          |
| Finalize PDF 1                   | Code                  | Finalizes PDF 1 for email attachment              | Convert HTML to PDF 1              | Merge Documents                             |                                                                                               |
| Finalize PDF 2                   | Code                  | Finalizes PDF 2 for email attachment              | Convert HTML to PDF 2              | Merge Documents                             |                                                                                               |
| Merge Documents                  | Merge                  | Merges two PDFs if both translations exist       | Finalize PDF 1, Finalize PDF 2     | Prepare Email                               |                                                                                               |
| Prepare Email                   | Code                  | Prepares email content including attachments     | Merge Documents                   | Cost – Build record                         |                                                                                               |
| Cost – Build record              | Code                  | Calculates translation and conversion costs      | Prepare Email                     | Data Table, Send Two Attachments?           |                                                                                               |
| Data Table                      | Data Table             | Displays cost and usage information               | Cost – Build record               | None                                        |                                                                                               |
| Send Two Attachments?            | If                     | Determines email attachment count                  | Cost – Build record               | Send Email (2 PDFs), Send Email(1 PDF)      |                                                                                               |
| Send Email (2 PDFs)              | Gmail                  | Sends email with two PDF attachments               | Send Two Attachments?             | None                                        | Requires Gmail OAuth2 (Send) credential connected in n8n                                    |
| Send Email(1 PDF)               | Gmail                  | Sends email with one PDF attachment                | Send Two Attachments?             | None                                        | Requires Gmail OAuth2 (Send) credential connected in n8n                                    |
| Sticky Note (various)            | Sticky Note            | Notes and comments for user reference             | N/A                             | N/A                                         | Several sticky notes present but no content provided                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure webhook endpoint with unique ID  
   - Configure form to accept PDF file and two language selections (Language 1 required, Language 2 optional)  

2. **Create Code Node "Process Form Data"**  
   - Type: Code  
   - Write JavaScript to parse incoming form data, extract PDF binary, and normalize language parameters  
   - Connect output of Form Trigger to this node  

3. **Create Extract From File Node "Extract Text from PDF"**  
   - Input: Processed PDF from previous node  
   - Configure to extract raw text from the PDF document  
   - Connect Process Form Data output to this node  

4. **Create Code Node "Convert to HTML"**  
   - Write JS code to transform extracted text into well-structured HTML with CSS styles to preserve formatting  
   - Connect Extract Text from PDF output to this node  

5. **Create Google Translate Node "Translate to Language 1"**  
   - Credentials: Add Google Service Account with enabled Translation API  
   - Input: HTML content from Convert to HTML  
   - Configure to translate from English to Language 1, preserving HTML tags  
   - Connect Convert to HTML output to this node  

6. **Create If Node "Has 2nd Language?"**  
   - Condition: Check if Language 2 is set (not empty) in the form data  
   - Connect Convert to HTML output also to this node (to allow second translation branch)  

7. **Create Google Translate Node "Translate to Language 2"**  
   - Same configuration as Language 1 node, target Language 2  
   - Connect If node’s true branch to this node  

8. **Create Code Nodes "Prepare HTML 1" and "Prepare HTML 2"**  
   - Convert translated HTML output into binary format for ConvertAPI  
   - Connect Translate to Language 1 to Prepare HTML 1  
   - Connect Translate to Language 2 to Prepare HTML 2  

9. **Create HTTP Request Nodes "Convert HTML to PDF 1" and "Convert HTML to PDF 2"**  
   - Method: POST to ConvertAPI PDF conversion endpoint  
   - Add ConvertAPI secret as query parameter or credential  
   - Input: Binary HTML from Prepare HTML nodes  
   - Connect Prepare HTML 1 to Convert HTML to PDF 1  
   - Connect Prepare HTML 2 to Convert HTML to PDF 2  

10. **Create Code Nodes "Finalize PDF 1" and "Finalize PDF 2"**  
    - Prepare PDF binary with correct mime type and metadata for email attachment  
    - Connect Convert API PDF nodes to corresponding Finalize PDF nodes  

11. **Create Merge Node "Merge Documents"**  
    - Merge both finalized PDFs into one stream (or pass single PDF if only one language)  
    - Connect outputs of Finalize PDF 1 and Finalize PDF 2 to Merge Documents  

12. **Create Code Node "Prepare Email"**  
    - Build email subject, recipients, body, and attach merged PDFs  
    - Connect output of Merge Documents to this node  

13. **Create Code Node "Cost – Build record"**  
    - Calculate cost of translation and PDF conversion based on API usage and document size  
    - Connect Prepare Email output to this node  

14. **Create Data Table Node "Data Table"**  
    - Display or log cost records for review  
    - Connect Cost – Build record output to this node  

15. **Create If Node "Send Two Attachments?"**  
    - Check if two PDFs are to be sent (based on presence of second language translation)  
    - Connect Cost – Build record output to this node  

16. **Create Gmail Nodes "Send Email (2 PDFs)" and "Send Email(1 PDF)"**  
    - Configure Gmail OAuth2 credentials with send permission  
    - Configure email parameters to attach one or two PDFs accordingly  
    - Connect If node’s true branch to "Send Email (2 PDFs)"  
    - Connect If node’s false branch to "Send Email(1 PDF)"  

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Google Translate nodes require connecting a Google Service Account credential with Translation API enabled in n8n.       | Google Cloud Console -> Service Accounts -> API & Service activation                                                   |
| ConvertAPI nodes require a ConvertAPI secret either as query param "Secret" or stored credential in n8n.                  | https://www.convertapi.com/doc                                                                                         |
| Gmail nodes require OAuth2 credentials with send mail permission enabled in n8n.                                          | Setup Gmail OAuth2 in n8n credentials                                                                                   |
| The workflow preserves HTML tags during translation to maintain formatting in the final PDFs.                             | Important for layout preservation during automated translation and PDF regeneration                                     |
| The workflow conditionally handles presence of a second language for translation and emailing.                          | Ensures flexibility if only one language translation is requested                                                      |
| Cost tracking calculates usage for API calls and conversion processes for transparency and billing purposes.             | Useful for monitoring and controlling API costs                                                                        |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.