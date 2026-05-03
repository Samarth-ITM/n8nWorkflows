Automated Invoice Processing & Filing with IMAP, AI, Google Drive & DateV

https://n8nworkflows.xyz/workflows/automated-invoice-processing---filing-with-imap--ai--google-drive---datev-9439


# Automated Invoice Processing & Filing with IMAP, AI, Google Drive & DateV

---

### 1. Workflow Overview

This workflow automates the entire process of invoice management starting from receiving invoice emails to filing and accounting integration. It targets bookkeeping and accounting teams or automated financial departments that handle incoming invoices via email, aiming to reduce manual data entry and improve accuracy.

**Target Use Cases:**

- Automatically processing incoming invoice emails with PDF attachments.
- Extracting and structuring invoice data using AI to ensure standardized bookkeeping.
- Storing invoices in organized Google Drive folders by month.
- Logging invoice data into Google Sheets for tracking.
- Sending processed invoices to the DateV accounting system via email.
- Archiving processed emails to maintain inbox organization.

**Logical Blocks:**

- **1.1 Input Reception:** Capturing incoming emails and extracting attachments.
- **1.2 Attachment Preparation:** Splitting and uploading attachments, making files accessible.
- **1.3 Invoice Text Extraction:** Extracting textual content from PDF invoices.
- **1.4 AI Data Extraction & Formatting:** Using AI to parse invoice data into structured JSON.
- **1.5 Date Metadata Enrichment:** Adding human-readable date components and formats.
- **1.6 Temporary Storage Upload:** Storing files temporarily on Google Drive.
- **1.7 Folder Search & Organization:** Locating or creating Google Drive folders by invoice month.
- **1.8 Final Upload & Naming:** Renaming and moving invoices to the correct folder.
- **1.9 Data Logging:** Appending structured invoice data into Google Sheets.
- **1.10 Accounting Integration:** Sending invoices by email to DateV.
- **1.11 Email Archiving:** Moving processed emails to a designated archive folder.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Listens for incoming emails via IMAP and extracts all attachments for processing.

**Nodes Involved:**  
- Email Trigger (IMAP)  
- Code in JavaScript  

**Node Details:**  

- **Email Trigger (IMAP)**  
  - Type: Email Read (IMAP)  
  - Role: Listens for new emails in the inbox and retrieves attachments.  
  - Configuration: Uses default IMAP connection, format resolved, no specific filters.  
  - Inputs: None (trigger node).  
  - Outputs: Email data including attachments.  
  - Edge Cases: Connection/authentication errors; emails without attachments; large attachments causing timeouts.

- **Code in JavaScript**  
  - Type: Code  
  - Role: Splits multiple attachments into individual items with one binary file per item named `data`.  
  - Key Code Logic: Iterates over all binary keys in the input item, creates new items each containing one binary attachment under `data`.  
  - Inputs: Email data with multiple attachments.  
  - Outputs: Multiple items for single attachments.  
  - Edge Cases: Emails with no attachments (empty output); binary property naming inconsistencies.

---

#### 1.2 Attachment Preparation

**Overview:**  
Uploads each attachment to a temporary Google Drive folder, ensuring availability for all further steps.

**Nodes Involved:**  
- Loop Over Items  
- Upload file  

**Node Details:**  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each attachment item individually.  
  - Inputs: Individual attachment items from previous node.  
  - Outputs: One attachment per iteration.  
  - Edge Cases: Batch size configuration might affect performance.

- **Upload file**  
  - Type: Google Drive File Upload  
  - Role: Uploads the individual attachment file to a designated Incoming Files folder on Google Drive.  
  - Configuration: Uses binary property `attachment_0` file name, uploads to a specified folder ID.  
  - Inputs: Single attachment file.  
  - Outputs: Metadata about uploaded file including file ID.  
  - Edge Cases: Drive API quota limits; folder permissions; filename conflicts.

---

#### 1.3 Invoice Text Extraction

**Overview:**  
Downloads the temporarily uploaded PDF, extracts visible text for AI processing.

**Nodes Involved:**  
- Search Month Folder  
- Download file  
- Extract From PDF  

**Node Details:**  

- **Search Month Folder**  
  - Type: Google Drive File/Folder Search  
  - Role: Finds a Google Drive folder for the current month (e.g., "October") to organize invoices.  
  - Configuration: Uses a query on folder name matching the current month in German locale.  
  - Inputs: None (triggered by previous upload).  
  - Outputs: Folder metadata including webViewLink.  
  - Edge Cases: Folder not found returns empty; permissions issues.

- **Download file**  
  - Type: Google Drive File Download  
  - Role: Downloads the PDF file using previously obtained file ID.  
  - Inputs: File ID from Upload file node.  
  - Outputs: Binary data of the PDF.  
  - Edge Cases: File not found; download failures.

- **Extract From PDF**  
  - Type: Extract From File  
  - Role: Extracts all visible text content from the PDF binary data.  
  - Configuration: Operates on binary property named `data`.  
  - Inputs: Binary PDF file.  
  - Outputs: Extracted text as JSON.  
  - Edge Cases: Complex PDF layouts may reduce extraction accuracy; large files may timeout.

---

#### 1.4 AI Data Extraction & Formatting

**Overview:**  
Uses an OpenAI GPT-4o model to parse unstructured invoice text into a structured JSON object with invoice fields.

**Nodes Involved:**  
- Extract & Format Data  

**Node Details:**  

- **Extract & Format Data**  
  - Type: OpenAI via LangChain Node  
  - Role: Sends extracted invoice text to AI model with instructions to output a strict JSON of key invoice data fields.  
  - Configuration: Model GPT-4o, prompt includes detailed instructions for German number formatting (comma decimal), mandatory fields, fallback rules, and example output.  
  - Key Expression:  
    - `{{ $json.text }}` injects extracted PDF text into the prompt.  
  - Inputs: Extracted text JSON.  
  - Outputs: JSON object containing fields like company, invoiceNumber, invoiceDate, amounts, currency, taxRatePercent.  
  - Edge Cases: AI misinterpretation; incomplete data; API rate limits; malformed JSON outputs.

---

#### 1.5 Date Metadata Enrichment

**Overview:**  
Augments the extracted data with additional date formats and metadata for foldering and naming conventions.

**Nodes Involved:**  
- Get Additional Date Data  

**Node Details:**  

- **Get Additional Date Data**  
  - Type: Code  
  - Role: Parses the invoiceDate field into German date format (DD.MM.YYYY), extracts month name (in German), year, and day.  
  - Key Logic: Uses JavaScript Date object and locale formatting ('de-DE').  
  - Inputs: AI-extracted invoice data JSON.  
  - Outputs: Extended JSON with fields `datumDeutsch`, `monat`, and `jahr`.  
  - Edge Cases: Invalid or empty date string causing parsing errors.

---

#### 1.6 Temporary Storage Upload

**Overview:**  
Uploads the original invoice PDF again to a temporary Google Drive folder to ensure accessibility in subsequent workflow steps.

**Nodes Involved:**  
- Upload file (already detailed in 1.2)  
- (Implicit in the flow, this step uses the earlier Upload file node.)

**Note:** This block overlaps with 1.2 as the temporary storage upload is done as soon as attachments are split.

---

#### 1.7 Folder Search & Organization

**Overview:**  
Locates the correct monthly folder in Google Drive to organize invoices by month and year.

**Nodes Involved:**  
- Search Month Folder (already detailed in 1.3)  

**Note:** Folder search uses current month name and is critical for correct file placement.

---

#### 1.8 Final Upload & Naming

**Overview:**  
Downloads the temporarily stored file and uploads it again to the target monthly folder with a clean, standardized name based on invoice data.

**Nodes Involved:**  
- Download file3  
- Upload file1  
- Loop Over Items (to handle multiple files)

**Node Details:**  

- **Download file3**  
  - Type: Google Drive File Download  
  - Role: Downloads the temporarily stored invoice PDF by file ID.  
  - Inputs: File ID from Upload file node.  
  - Outputs: Binary PDF data.  
  - Edge Cases: Download failure; file missing.

- **Upload file1**  
  - Type: Google Drive File Upload  
  - Role: Uploads PDF to the monthly folder with filename formatted as `{invoiceDate}_{company}_{invoiceNumber}.pdf` (spaces replaced by hyphens).  
  - Configuration: Uses folderId from Search Month Folder node's `webViewLink` (converted to folder ID).  
  - Inputs: Binary PDF and metadata.  
  - Outputs: Metadata of newly uploaded file.  
  - Edge Cases: Filename conflicts; Drive permission issues.

- **Loop Over Items**  
  - Used again to process files individually.

---

#### 1.9 Data Logging

**Overview:**  
Appends the extracted and formatted invoice data as a new row in a Google Sheets document for bookkeeping and tracking.

**Nodes Involved:**  
- Google Sheets  

**Node Details:**  

- **Google Sheets**  
  - Type: Google Sheets Append  
  - Role: Adds a new row with invoice data fields: Company, Description, Invoice Number, Invoice Date (month name), Net Amount, Tax Rate Percent, Tax Amount, Gross Amount, Currency.  
  - Configuration: Fixed schema mapping each field explicitly, prevents type conversion.  
  - Inputs: Extended invoice JSON data.  
  - Outputs: Confirmation of sheet append.  
  - Edge Cases: Sheet access permissions; rate limits; schema mismatch.

---

#### 1.10 Accounting Integration

**Overview:**  
Sends the finalized invoice PDF as an email attachment to the DateV accounting inbox.

**Nodes Involved:**  
- Download file1  
- Send to DateV  

**Node Details:**  

- **Download file1**  
  - Type: Google Drive File Download  
  - Role: Downloads the finalized invoice PDF for emailing.  
  - Inputs: File ID from Upload file node.  
  - Outputs: Binary PDF data.  
  - Edge Cases: Missing files; API issues.

- **Send to DateV**  
  - Type: Email Send  
  - Role: Sends an email with subject "Beleg Rechnung" to the DateV inbox address with the invoice attached.  
  - Configuration: Fixed to/from email addresses (placeholders intended to be replaced).  
  - Inputs: Binary PDF attachment.  
  - Outputs: Email send status.  
  - Edge Cases: SMTP authentication errors; invalid recipient; attachment size limits.

---

#### 1.11 Email Archiving

**Overview:**  
Moves the processed email from the inbox to an archive folder to keep the mailbox organized.

**Nodes Involved:**  
- Download file2  
- MoveEmail email  
- Download file3 (for final upload)  

**Node Details:**  

- **Download file2**  
  - Downloads the file to be used downstream.  
  - Inputs: File ID.  
  - Outputs: Binary data.

- **MoveEmail email**  
  - Type: IMAP operation  
  - Role: Moves the processed email from INBOX to folder "Rechnungen_DDB".  
  - Configuration: Uses UID from Email Trigger (IMAP) node.  
  - Inputs: Email UID.  
  - Outputs: Operation result.  
  - Edge Cases: IMAP folder permission errors; UID missing or invalid.

- **Download file3**  
  - Used to trigger final upload after moving email.  

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                           | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                 |
|-----------------------|----------------------------|-----------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------|
| Email Trigger (IMAP)   | Email Read (IMAP)           | Detect incoming emails and attachments  | None                          | Code in JavaScript           | Receive IMAP-Mail & Extract Attachments from Email                         |
| Code in JavaScript     | Code                       | Split multiple attachments into items   | Email Trigger (IMAP)           | Loop Over Items              | Receive IMAP-Mail & Extract Attachments from Email                         |
| Loop Over Items        | SplitInBatches             | Process each attachment individually    | Code in JavaScript             | Upload file (on second output) | Upload to Temporary Storage                                                |
| Upload file            | Google Drive Upload        | Upload attachments temporarily          | Loop Over Items                | Search Month Folder          | Upload to Temporary Storage                                                |
| Search Month Folder    | Google Drive Search        | Find monthly folder for invoice storage | Upload file                   | Download file                |                                                                            |
| Download file          | Google Drive Download      | Download temporarily stored PDF          | Search Month Folder            | Extract From PDF             |                                                                            |
| Extract From PDF       | Extract From File          | Extract text from PDF                     | Download file                 | Extract & Format Data        | Extract & Log Invoice Data                                                 |
| Extract & Format Data  | OpenAI LangChain           | AI-based structured invoice data extraction | Extract From PDF              | Get Additional Date Data     | Extract & Log Invoice Data                                                 |
| Get Additional Date Data | Code                     | Add human-readable dates and metadata    | Extract & Format Data          | Google Sheets                | Extract & Log Invoice Data                                                 |
| Google Sheets          | Google Sheets Append       | Log invoice data into spreadsheet        | Get Additional Date Data       | Download file1               | Extract & Log Invoice Data                                                 |
| Download file1         | Google Drive Download      | Download file for sending to DateV       | Google Sheets                 | Send to DateV                | Send to DateV                                                             |
| Send to DateV          | Email Send                 | Email invoice to DateV accounting         | Download file1                | Download file2               | Send to DateV                                                             |
| Download file2         | Google Drive Download      | Download file for email archiving steps   | Send to DateV                 | MoveEmail email             |                                                                            |
| MoveEmail email        | IMAP Move Email            | Move processed email to archive folder    | Download file2                | Download file3               | Move Email to Archive/Folder                                               |
| Download file3         | Google Drive Download      | Download file for final upload             | MoveEmail email              | Upload file1                 | Upload to final destination                                                |
| Upload file1           | Google Drive Upload        | Upload renamed invoice to monthly folder  | Download file3               | Loop Over Items              | Upload to final destination                                                |
| Sticky Note            | Sticky Note                | Documentation                            | N/A                           | N/A                         | Extract & Log Invoice Data Steps described                                |
| Sticky Note1           | Sticky Note                | Documentation                            | N/A                           | N/A                         | Send to DateV Step details                                                |
| Sticky Note2           | Sticky Note                | Documentation                            | N/A                           | N/A                         | Move Email to Archive/Folder Step                                         |
| Sticky Note3           | Sticky Note                | Documentation                            | N/A                           | N/A                         | Upload to Temporary Storage Steps                                         |
| Sticky Note4           | Sticky Note                | Documentation                            | N/A                           | N/A                         | Receive IMAP-Mail & Extract Attachments from Email                        |
| Sticky Note5           | Sticky Note                | Documentation                            | N/A                           | N/A                         | Workflow Overview Summary                                                |
| Sticky Note6           | Sticky Note                | Documentation                            | N/A                           | N/A                         | Upload to final destination steps                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Email Trigger (IMAP) node**  
   - Type: Email Read (IMAP)  
   - Configure credentials for your IMAP email account.  
   - Set to trigger on new emails, format resolved.  
   - No special filters needed.

2. **Create Code node ("Code in JavaScript")**  
   - Purpose: Split multiple attachments into individual items.  
   - Use the following JS snippet to iterate over all binary attachments and output one item per attachment with binary property named `data`.

3. **Create SplitInBatches node ("Loop Over Items")**  
   - Purpose: Process each attachment one by one in a controlled loop.

4. **Create Google Drive Upload node ("Upload file")**  
   - Purpose: Upload each attachment to a temporary "Incoming Files" folder in Google Drive.  
   - Configure credentials for Google Drive.  
   - Set folder ID for the temporary storage folder.  
   - Use binary property `attachment_0` for upload file name.

5. **Create Google Drive Search node ("Search Month Folder")**  
   - Purpose: Find the folder matching the current month (e.g., "Oktober").  
   - Use query string with current month name in German locale (e.g., `={{ $now.setLocale('de').toFormat('MMMM') }}`).  
   - Configure credentials.

6. **Create Google Drive Download node ("Download file")**  
   - Purpose: Download the temporarily uploaded PDF file by file ID.  
   - Link file ID from Upload file node.

7. **Create Extract From File node ("Extract From PDF")**  
   - Purpose: Extract all visible text content from the PDF binary.  
   - Use binary property `data`.

8. **Create OpenAI LangChain node ("Extract & Format Data")**  
   - Purpose: Send extracted text to AI model for structured invoice data extraction.  
   - Use model GPT-4o (or your preferred GPT-4 variant).  
   - Configure prompt to parse invoice data into JSON with fields: company, invoiceNumber, invoiceDate, description, netAmount, taxRatePercent, taxAmount, grossAmount, currency.  
   - Ensure decimal separator is a comma and currency symbol is € or $ as per instructions.

9. **Create Code node ("Get Additional Date Data")**  
   - Purpose: Parse invoiceDate field to generate German date format (DD.MM.YYYY), month name (German), and year as strings.  
   - Use JavaScript Date and locale formatting.

10. **Create Google Sheets Append node ("Google Sheets")**  
    - Purpose: Append invoice data to a Google Sheet.  
    - Configure credentials and spreadsheet ID.  
    - Define columns: Company, Description, Invoice Number, Invoice Date (month name), Net Amount, Tax Rate Percent, Tax Amount, Gross Amount, Currency.  
    - Map fields from previous node JSON.

11. **Create Google Drive Download node ("Download file1")**  
    - Purpose: Download the file for sending to DateV.

12. **Create Email Send node ("Send to DateV")**  
    - Purpose: Send an email with the invoice attached to your DateV inbox.  
    - Configure SMTP or email credentials.  
    - Set To to your DateV inbox email address.  
    - Set From appropriately.  
    - Subject: "Beleg Rechnung".  
    - Attach the downloaded PDF binary.

13. **Create Google Drive Download node ("Download file2")**  
    - Purpose: Prepare the file for email archiving steps.

14. **Create IMAP Move Email node ("MoveEmail email")**  
    - Purpose: Move processed email from INBOX to archive folder "Rechnungen_DDB".  
    - Use email UID from Email Trigger.  
    - Configure IMAP credentials.

15. **Create Google Drive Download node ("Download file3")**  
    - Purpose: Download the file for final upload to monthly folder.

16. **Create Google Drive Upload node ("Upload file1")**  
    - Purpose: Upload the invoice PDF to the monthly folder with a standardized filename.  
    - Filename format: `{invoiceDate}_{company}_{invoiceNumber}.pdf` with spaces replaced by hyphens.  
    - Folder ID: taken from Search Month Folder node's output (convert webViewLink to folder ID if needed).  
    - Use binary data of the PDF.

17. **Connect nodes following this order:**  
    - Email Trigger → Code in JavaScript → Loop Over Items → Upload file → Search Month Folder → Download file → Extract From PDF → Extract & Format Data → Get Additional Date Data → Google Sheets → Download file1 → Send to DateV → Download file2 → MoveEmail email → Download file3 → Upload file1 → Loop Over Items (to process next attachment if any).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow strictly formats numbers in German style using commas as decimal separators and uses € or $ currency symbols instead of text codes like EUR or USD. | Important for correct bookkeeping formats and compliance with German accounting standards.      |
| Ensure your DateV inbox email address is correctly configured in the "Send to DateV" node for seamless accounting integration.                                    | Replace placeholder emails in the workflow.                                                    |
| For Google Drive folder queries, the month names are localized to German (e.g., "Oktober").                                                                        | This requires n8n environment or server locale support for 'de' locale.                         |
| API rate limits and connection timeouts may occur during bulk invoice processing; consider adding error handling or retries.                                      | General best practice for production workflows involving external APIs.                        |
| Workflow documentation sticky notes are embedded in the workflow for clarity and can serve as inline help during maintenance or modification.                   | Useful for team collaboration and onboarding new users to the workflow.                        |

---

**Disclaimer:**  
The text provided originates solely from an automated workflow created with n8n, an integration and automation tool. This processing fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---