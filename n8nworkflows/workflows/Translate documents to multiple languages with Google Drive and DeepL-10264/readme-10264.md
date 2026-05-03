Translate documents to multiple languages with Google Drive and DeepL

https://n8nworkflows.xyz/workflows/translate-documents-to-multiple-languages-with-google-drive-and-deepl-10264


# Translate documents to multiple languages with Google Drive and DeepL

### 1. Workflow Overview

This workflow automates the translation of documents uploaded to a specific Google Drive folder into multiple target languages using the DeepL API. It supports common document file types (PDF, DOCX including Google Docs, TXT, Markdown), extracting their text content, translating it, and then saving the translated files back to Google Drive. Optionally, it can log translation activities into a Notion database and send notification emails upon completion or errors.

**Target Use Cases:**
- Multilingual document translation automation for teams or individuals.
- Centralized handling of document translations triggered by new file uploads.
- Integration of translation workflow with Google Drive, DeepL, Gmail, and Notion.

**Logical Blocks:**

- **1.1 Input Reception & Configuration:** Detects new files in a Google Drive folder and merges user-defined configuration.
- **1.2 File Format Detection & Text Extraction:** Identifies file type and extracts text accordingly.
- **1.3 Translation Loop:** Splits target languages and processes translation, filename generation, file creation, and saving for each language.
- **1.4 Completion & Notification:** Aggregates results, optionally logs to Notion, and sends notification emails.
- **1.5 Error Handling (disabled):** Code and notification nodes for error handling, currently disabled.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration

**Overview:**  
Monitors for new file creation events in a specified Google Drive folder and loads user-defined configuration parameters for the workflow.

**Nodes Involved:**  
- Google Drive Trigger  
- Configuration (Edit Here)  
- Merge Configuration  

**Node Details:**

- **Google Drive Trigger**  
  - Type: Google Drive Trigger  
  - Role: Listens for newly created files in a specified Drive folder every minute.  
  - Config: Watches a specific folder (user must replace `YOUR_SOURCE_FOLDER_ID_HERE` with actual folder ID).  
  - Inputs: None (trigger node)  
  - Outputs: Emits metadata of created files.  
  - Failure Modes: Folder ID misconfiguration, Google Drive API auth issues, network timeouts.

- **Configuration (Edit Here)**  
  - Type: Set  
  - Role: Holds user editable settings such as source/destination folder IDs, target languages array, notification email, Notion integration toggles/IDs.  
  - Config: Values to be customized (e.g., `targetLanguages` default to `["EN", "ZH", "KO", "ES", "FR", "DE"]`).  
  - Inputs: None  
  - Outputs: Configuration object used downstream.  
  - Failure Modes: Misconfigured or missing values lead to workflow failure or unexpected behavior.

- **Merge Configuration**  
  - Type: Merge  
  - Role: Combines the triggered file metadata and the configuration parameters into a single data object for downstream processing.  
  - Inputs: From Google Drive Trigger and Configuration node  
  - Outputs: Single combined JSON object  
  - Failure Modes: Data mismatch or missing inputs.

---

#### 1.2 File Format Detection & Text Extraction

**Overview:**  
Detects the file format of the uploaded document and routes it to the appropriate text extraction node based on MIME type or file extension.

**Nodes Involved:**  
- Detect File Format (Switch)  
- Download PDF  
- Download DOCX  
- Download TXT  
- Download Markdown  
- Extract Text from PDF  
- Extract Text from DOCX  
- Read TXT File  
- Read Markdown File  
- Format Text Data  

**Node Details:**

- **Detect File Format**  
  - Type: Switch  
  - Role: Routes data based on MIME type or filename extension to one of four categories: PDF, DOCX/Google Docs, TXT, Markdown.  
  - Config: Checks MIME types such as `application/pdf` for PDF, Google Docs format, plain text, and markdown extensions.  
  - Inputs: Combined data from previous block  
  - Outputs: Up to 4 outputs, one per format type  
  - Failure Modes: Unrecognized MIME types cause no routing; missing or unexpected MIME data.

- **Download PDF / DOCX / TXT / Markdown**  
  - Type: Google Drive (Download)  
  - Role: Downloads the file binary from Google Drive for extraction.  
  - Config: Uses file ID from input JSON. PDF and DOCX have conversion options (PDF conversion empty, DOCX conversion empty but ready). TXT and Markdown download directly.  
  - Inputs: From Detect File Format outputs  
  - Outputs: Binary file data for extraction  
  - Failure Modes: File not found, permissions errors, API rate limits.

- **Extract Text from PDF**  
  - Type: Extract From File  
  - Role: Extracts text from the PDF binary file.  
  - Config: Operation set to PDF extraction.  
  - Inputs: Download PDF node  
  - Outputs: Extracted text in JSON key.  
  - Failure Modes: OCR failures with scanned PDFs, corrupted files.

- **Extract Text from DOCX**  
  - Type: Extract From File  
  - Role: Extracts text from DOCX binary files.  
  - Config: Operation set to text extraction.  
  - Inputs: Download DOCX node  
  - Outputs: Extracted text in JSON key `text`.  
  - Failure Modes: Complex DOCX structure may lose formatting or miss content.

- **Read TXT File / Read Markdown File**  
  - Type: Extract From File  
  - Role: Reads plain text content from TXT or Markdown files.  
  - Config: Operation set to text extraction.  
  - Inputs: Download TXT / Download Markdown node  
  - Outputs: Extracted text in JSON key `text`.  
  - Failure Modes: Encoding issues.

- **Format Text Data**  
  - Type: Set  
  - Role: Consolidates extracted text and metadata (original filename, extension, target languages, folder IDs, email, Notion flags) into a structured JSON object for translation steps.  
  - Inputs: Text extraction nodes outputs  
  - Outputs: Structured JSON with all necessary info  
  - Failure Modes: Missing fields or incorrect value assignments.  

---

#### 1.3 Translation Loop

**Overview:**  
Splits the list of target languages into separate items and, for each language, translates the extracted text, generates a new filename, creates a binary file, and uploads it to Google Drive.

**Nodes Involved:**  
- Split by Language  
- Translate a language  
- Generate Filename  
- Code in JavaScript  
- Create Translated File  
- Save to Google Drive  

**Node Details:**

- **Split by Language**  
  - Type: Split Out  
  - Role: Splits workflow data into multiple items, one per target language from the `targetLanguages` array.  
  - Inputs: Formatted text data with target languages array  
  - Outputs: One item per language  
  - Failure Modes: Empty or invalid targetLanguages array.

- **Translate a language**  
  - Type: DeepL  
  - Role: Calls DeepL API to translate the extracted text to the current language.  
  - Config: Source text from `extractedText`, target language from current split item.  
  - Inputs: One language item from Split by Language  
  - Outputs: Translated text per language  
  - Failure Modes: API quota exceeded, network errors, invalid target language codes.

- **Generate Filename**  
  - Type: Set  
  - Role: Prepares the translated text, language code, and new filename string (e.g., original filename plus language suffix).  
  - Inputs: From Translate a language node  
  - Outputs: JSON including `translatedText`, `languageCode`, and `newFilename`  
  - Failure Modes: Incorrect filename generation logic may cause overwrites or invalid filenames.

- **Code in JavaScript**  
  - Type: Code  
  - Role: Converts translated text into binary format suitable for file writing, preserving metadata.  
  - Inputs: From Generate Filename  
  - Outputs: JSON plus binary data property named `data`  
  - Failure Modes: Buffer encoding issues, large texts causing memory issues.

- **Create Translated File**  
  - Type: Write Binary File  
  - Role: Writes the translated text as a local binary file (temporary).  
  - Config: Filename from `newFilename`, binary data from previous node.  
  - Inputs: From Code in JavaScript  
  - Outputs: Binary file reference for upload  
  - Failure Modes: Disk write permissions, filename collisions.

- **Save to Google Drive**  
  - Type: Google Drive  
  - Role: Uploads the translated file to the destination folder in Google Drive.  
  - Config: Filename and folder ID dynamically set, Drive ID configured by user.  
  - Inputs: From Create Translated File  
  - Outputs: Google Drive file metadata including web view link  
  - Failure Modes: Upload failures, permission issues, folder ID misconfiguration.

---

#### 1.4 Completion & Notification

**Overview:**  
After all translations finish, aggregates all translation results, prepares a notification summary, optionally records into Notion, and sends an email notification.

**Nodes Involved:**  
- Check if Notion Enabled  
- Record in Notion  
- Merge Results  
- Aggregate Translations  
- Prepare Notification  
- Send Gmail Notification  

**Node Details:**

- **Check if Notion Enabled**  
  - Type: If  
  - Role: Checks boolean flag from configuration whether to record translations in Notion.  
  - Inputs: From Save to Google Drive  
  - Outputs: Two branches: true (Notion enabled), false (skip Notion)  
  - Failure Modes: Misconfigured flag leads to skipping Notion.

- **Record in Notion**  
  - Type: Notion  
  - Role: Creates a new page in a specified Notion database with translation metadata (original filename, language code, timestamp, Google Drive link).  
  - Config: Database ID from configuration, properties dynamically set from JSON data.  
  - Inputs: True branch from Check if Notion Enabled  
  - Outputs: Notion API response  
  - Failure Modes: Notion API auth errors, invalid database ID, network errors.

- **Merge Results**  
  - Type: Merge  
  - Role: Combines Notion recording output (if any) with the rest of the translation results for final aggregation.  
  - Inputs: From Record in Notion (true) or directly from Save to Google Drive (false)  
  - Outputs: Combined data stream  
  - Failure Modes: Mismatched input counts.

- **Aggregate Translations**  
  - Type: Aggregate  
  - Role: Collects all individual translation items into one dataset for notification summary.  
  - Inputs: From Merge Results  
  - Outputs: Aggregated JSON array  
  - Failure Modes: Large result sets may cause memory issues.

- **Prepare Notification**  
  - Type: Set  
  - Role: Constructs a summary object including timestamp, original filename, total and translated languages for the notification email.  
  - Inputs: From Aggregate Translations and configuration nodes  
  - Outputs: JSON with summary data  
  - Failure Modes: Missing input data.

- **Send Gmail Notification**  
  - Type: Gmail  
  - Role: Sends an HTML email to the configured notification email address with translation summary details.  
  - Config: Uses OAuth2 Gmail credentials, dynamic email and subject content from summary.  
  - Inputs: From Prepare Notification  
  - Outputs: Email send confirmation  
  - Failure Modes: Gmail auth errors, invalid email address, quota limits.

---

#### 1.5 Error Handling (Disabled)

**Overview:**  
Contains a JavaScript code node and Gmail node designed for error capturing and notification, currently disabled.

**Nodes Involved:**  
- Error Handler (Code)  
- Send Error Notification (Gmail)  

**Node Details:**

- **Error Handler**  
  - Type: Code  
  - Role: Logs error details including timestamp, error message, file name, and step where error occurred.  
  - Status: Disabled  
  - Failure Modes: Disabled, no effect unless enabled.

- **Send Error Notification**  
  - Type: Gmail  
  - Role: Sends plain text email with error details to notification email.  
  - Status: Disabled  
  - Failure Modes: Disabled, no effect unless enabled.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                                  | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                   |
|-------------------------|-----------------------|-------------------------------------------------|---------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Google Drive Trigger     | Google Drive Trigger  | Trigger on new files in source folder           | None                            | Merge Configuration, Configuration (Edit Here) |                                                                                              |
| Configuration (Edit Here)| Set                   | User settings (folders, languages, email, Notion)| None                            | Merge Configuration              | ## 1️⃣ Configuration: Edit this node to customize folders, languages, email, Notion           |
| Merge Configuration     | Merge                 | Combine trigger data and user configuration     | Google Drive Trigger, Configuration (Edit Here) | Detect File Format               |                                                                                              |
| Detect File Format       | Switch                | Route files by MIME type/file extension          | Merge Configuration             | Download PDF, DOCX, TXT, Markdown | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Download PDF             | Google Drive          | Download PDF file binary                          | Detect File Format (PDF)        | Extract Text from PDF            | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Download DOCX            | Google Drive          | Download DOCX/Google Doc file binary             | Detect File Format (DOCX)       | Extract Text from DOCX           | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Download TXT             | Google Drive          | Download TXT file binary                          | Detect File Format (TXT)        | Read TXT File                   | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Download Markdown        | Google Drive          | Download Markdown file binary                     | Detect File Format (Markdown)   | Read Markdown File              | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Extract Text from PDF    | Extract From File     | Extract text content from PDF                     | Download PDF                   | Format Text Data                | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Extract Text from DOCX   | Extract From File     | Extract text content from DOCX                     | Download DOCX                  | Format Text Data                | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Read TXT File            | Extract From File     | Extract text from TXT                             | Download TXT                   | Format Text Data                | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Read Markdown File       | Extract From File     | Extract text from Markdown                        | Download Markdown              | Format Text Data                | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Format Text Data         | Set                   | Consolidate extracted text and metadata          | Extract Text from PDF/DOCX/TXT/Markdown | Split by Language              | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Split by Language        | Split Out             | Split workflow by each target language           | Format Text Data               | Translate a language            | ## 3️⃣ Translation Loop                                                                      |
| Translate a language     | DeepL                 | Translate text to current language                | Split by Language              | Generate Filename              | ## 3️⃣ Translation Loop                                                                      |
| Generate Filename        | Set                   | Prepare translated text, language code, filename | Translate a language           | Code in JavaScript             | ## 3️⃣ Translation Loop                                                                      |
| Code in JavaScript       | Code                  | Convert translated text to binary                 | Generate Filename             | Create Translated File         | ## 3️⃣ Translation Loop                                                                      |
| Create Translated File   | Write Binary File      | Write binary file for upload                       | Code in JavaScript            | Save to Google Drive           | ## 3️⃣ Translation Loop                                                                      |
| Save to Google Drive     | Google Drive          | Upload translated file to destination folder      | Create Translated File        | Check if Notion Enabled        | ## 3️⃣ Translation Loop                                                                      |
| Check if Notion Enabled  | If                    | Decide if Notion logging is enabled               | Save to Google Drive          | Record in Notion, Merge Results | ## 4️⃣ Completion & Notification                                                            |
| Record in Notion         | Notion                 | Log translation metadata in Notion database       | Check if Notion Enabled (true) | Merge Results                 | ## 4️⃣ Completion & Notification                                                            |
| Merge Results            | Merge                 | Merge Notion output or skip with translation data | Record in Notion / Check if Notion Enabled (false) | Aggregate Translations       | ## 4️⃣ Completion & Notification                                                            |
| Aggregate Translations   | Aggregate             | Collect all translations for summary               | Merge Results                | Prepare Notification           | ## 4️⃣ Completion & Notification                                                            |
| Prepare Notification     | Set                   | Prepare summary data for email notification        | Aggregate Translations       | Send Gmail Notification       | ## 4️⃣ Completion & Notification                                                            |
| Send Gmail Notification  | Gmail                 | Send email notification with translation summary  | Prepare Notification         | None                         | ## 4️⃣ Completion & Notification                                                            |
| Error Handler            | Code                  | Logs and formats error info (disabled)             | None                        | Send Error Notification (disabled) |                                                                                              |
| Send Error Notification  | Gmail                 | Email error notification (disabled)                 | Error Handler               | None                         |                                                                                              |
| Sticky Note              | Sticky Note           | Documentation of overall workflow purpose          | None                        | None                         | # 🌍 Multi-Language Document Translation Workflow ... (Full sticky note content preserved)   |
| Sticky Note1             | Sticky Note           | Configuration instructions                          | None                        | None                         | ## 1️⃣ Configuration instructions                                                          |
| Sticky Note2             | Sticky Note           | File format detection and extraction explanation   | None                        | None                         | ## 2️⃣ File Format Detection & Text Extraction                                              |
| Sticky Note3             | Sticky Note           | Translation loop explanation                        | None                        | None                         | ## 3️⃣ Translation Loop                                                                      |
| Sticky Note4             | Sticky Note           | Completion and notification explanation             | None                        | None                         | ## 4️⃣ Completion & Notification                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Google Drive Trigger node:**
   - Event: `fileCreated`
   - Poll Interval: every minute
   - Trigger on specific folder: set `folderToWatch` to your source folder ID (replace `YOUR_SOURCE_FOLDER_ID_HERE`)
   - Connect credentials for Google Drive OAuth2.

3. **Add a Set node named "Configuration (Edit Here)":**
   - Add string fields:
     - `sourceFolderId` = your source folder ID
     - `destinationFolderId` = your destination folder ID
     - `notificationEmail` = your email address
     - `notionDatabaseId` = your Notion database ID (optional)
   - Add boolean field:
     - `enableNotion` (true/false)
   - Add array field:
     - `targetLanguages` default to `["EN", "ZH", "KO", "ES", "FR", "DE"]`

4. **Add a Merge node "Merge Configuration":**
   - Mode: Combine by position
   - Connect Google Drive Trigger output as first input
   - Connect Configuration node output as second input

5. **Add a Switch node "Detect File Format":**
   - Add conditions based on MIME type or filename extension:
     - PDF: `mimeType == "application/pdf"`
     - DOCX: `mimeType == "application/vnd.openxmlformats-officedocument.wordprocessingml.document"` OR Google Docs mime type
     - TXT: `mimeType == "text/plain"`
     - Markdown: `mimeType == "text/markdown"` OR filename ends with `.md`
   - Connect Merge Configuration output to this node.

6. **Create four Google Drive nodes for download:**
   - Operations: `download`
   - File ID: expression from input JSON `id`
   - For PDF and DOCX, enable Google File Conversion (empty config)
   - Connect each output of Switch to corresponding download node.

7. **Add Extract From File nodes:**
   - For PDF: operation `pdf`
   - For DOCX, TXT, Markdown: operation `text`
   - Destination key: `text` except PDF uses default
   - Connect each download node output to corresponding extract node.

8. **Add a Set node "Format Text Data":**
   - Assign fields:
     - `extractedText`: extracted text from previous node
     - `originalFilename`: filename from input
     - `fileExtension`: extension or type
     - `targetLanguages`, `destinationFolderId`, `notificationEmail`, `enableNotion`, `notionDatabaseId` from configuration
   - Connect extract nodes outputs to this node.

9. **Add a Split Out node "Split by Language":**
   - Field to split out: `targetLanguages`
   - Connect Format Text Data output.

10. **Add a DeepL node "Translate a language":**
    - Text: `extractedText` from input
    - Translate To: current `targetLanguages` item
    - Connect Split by Language output.
    - Configure with DeepL API credentials.

11. **Add a Set node "Generate Filename":**
    - Assign:
      - `translatedText`: from DeepL output
      - `languageCode`: current language code
      - `newFilename`: compose filename, e.g., original name + "_" + language code + extension
    - Connect Translate node output.

12. **Add a Code node "Code in JavaScript":**
    - Convert `translatedText` to a UTF-8 Buffer binary property `data`.
    - Return JSON with `newFilename`, `translatedText`, and binary data.
    - Connect Generate Filename output.

13. **Add a Write Binary File node "Create Translated File":**
    - File name: use expression `newFilename`
    - Data property: `data`
    - Connect Code node output.

14. **Add a Google Drive node "Save to Google Drive":**
    - Operation: upload file
    - Name: `newFilename`
    - Folder ID: `destinationFolderId` from configuration
    - Connect Write Binary File output.
    - Use Google Drive OAuth2 credentials.

15. **Add an If node "Check if Notion Enabled":**
    - Condition: boolean check if `enableNotion` is true
    - Connect Save to Google Drive output.

16. **Add a Notion node "Record in Notion":**
    - Resource: databasePage
    - Database ID: `notionDatabaseId` from config
    - Properties: map original filename, language code, timestamp, Google Drive web link dynamically
    - Connect If node "true" output.

17. **Add a Merge node "Merge Results":**
    - Mode: Combine by position
    - Connect Notion node and If node "false" output.

18. **Add an Aggregate node "Aggregate Translations":**
    - Aggregate all item data into one collection
    - Connect Merge Results output.

19. **Add a Set node "Prepare Notification":**
    - Create `summary` object with:
      - `timestamp` (current ISO)
      - `originalFile` (from format node)
      - `totalTranslations` (length of `targetLanguages`)
      - `translatedLanguages` (list from config)
    - Connect Aggregate node output.

20. **Add a Gmail node "Send Gmail Notification":**
    - To: `notificationEmail` from config
    - Subject: include original filename
    - HTML message with translation summary details
    - Connect Prepare Notification output.
    - Configure Gmail OAuth2 credentials.

21. **(Optional) Add error handling nodes:**
    - Code node to log errors
    - Gmail node to send error notifications
    - Connect error paths accordingly and enable if desired.

22. **Add Sticky Notes for documentation at appropriate positions.**

23. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| DeepL free tier limit: 500,000 characters/month                                                                                   | DeepL API usage limitation                                                                              |
| Supported file types: PDF, DOCX (including Google Docs), TXT, Markdown (.md)                                                      | Workflow input compatibility                                                                            |
| Default target languages: EN, ZH, KO, ES, FR, DE (customizable)                                                                    | Configuration node                                                                                      |
| Output files named with language codes appended, e.g., `document_en.pdf`                                                          | Output file naming convention                                                                            |
| Optional Notion integration for translation tracking                                                                               | Requires Notion API credentials and database ID                                                        |
| Gmail OAuth2 credentials needed for sending notification emails                                                                    | Email notification setup                                                                                 |
| Google Drive OAuth2 credentials required for trigger and file operations                                                           | Google Drive API authentication setup                                                                   |
| Workflow designed to process translation in parallel for all target languages                                                      | Performance consideration                                                                                |
| Workflow includes disabled error handling nodes that can be enabled for robust error notifications                                 | Error handling enhancement                                                                               |
| Sticky notes in workflow provide setup instructions and explanations in English and Japanese                                      | User guidance within the workflow                                                                        |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, a no-code automation tool. It strictly complies with content policies and contains no illegal or protected materials. All processed data are legal and public.