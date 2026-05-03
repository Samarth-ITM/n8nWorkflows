Extract Data from Thai Government Letters with Mistral OCR and Store in Google Sheets

https://n8nworkflows.xyz/workflows/extract-data-from-thai-government-letters-with-mistral-ocr-and-store-in-google-sheets-6702


# Extract Data from Thai Government Letters with Mistral OCR and Store in Google Sheets

---

### 1. Workflow Overview

This workflow automates the extraction of structured data from government letters received via two sources: LINE messaging and Google Drive. It performs OCR (Optical Character Recognition) on the documents using Mistral Document AI, extracts detailed information via OpenAI's language model, and logs the results in Google Sheets. Additionally, it responds back to LINE users confirming successful data capture or archives processed files from Google Drive.

**Target Use Cases:**  
- Automating document intake from LINE chat uploads and Google Drive folder monitoring  
- Extracting invoice or letter metadata in Thai government correspondence  
- Centralized logging of extracted data for further processing or auditing  
- User feedback via LINE and file management in Drive  

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving files from LINE webhook and Google Drive trigger  
- **1.2 File Download & Metadata Setup:** Downloading files and preparing metadata from each source  
- **1.3 Data Consolidation:** Merging inputs from LINE and Drive into a unified stream  
- **1.4 File Type Determination:** Differentiating between image and PDF for OCR processing  
- **1.5 OCR Processing with Mistral AI:** Extracting text from images and PDFs separately  
- **1.6 AI-based Information Extraction:** Parsing OCR results to structured JSON fields using OpenAI  
- **1.7 Data Logging:** Appending structured data to Google Sheets  
- **1.8 Post-Processing Actions:** Replying to LINE users or moving Google Drive files to archive  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block receives incoming documents either from a LINE bot webhook or a Google Drive folder trigger. It captures necessary identifiers for subsequent file download.

**Nodes Involved:**  
- LINE Webhook  
- Google Drive Trigger  

**Node Details:**  

- **LINE Webhook:**  
  - Type: Webhook (HTTP POST receiver)  
  - Configuration: Path `/webhook-linebot-test`, method POST to accept LINE Messaging API events  
  - Inputs: Incoming HTTP requests with LINE event payloads  
  - Outputs: Triggers downstream file download node  
  - Failure modes: Invalid webhook calls, missing or malformed payloads, authentication failures from LINE (handled externally)  
  - Credentials: None (public webhook endpoint)  

- **Google Drive Trigger:**  
  - Type: Google Drive Trigger node  
  - Configuration: Watches specific folder ID (environment variable `GDRIVE_INVOICE_FOLDER_ID`), triggers on new file creation every minute  
  - Inputs: Polls Drive for new files  
  - Outputs: File metadata for download  
  - Failure modes: API quota exceeded, folder ID misconfiguration, network errors  
  - Credentials: Google Drive OAuth2  

---

#### 2.2 File Download & Metadata Setup

**Overview:**  
Downloads the actual file content from LINE or Google Drive, and sets metadata fields for unified processing.

**Nodes Involved:**  
- Get Line File  
- Download File (Drive)  
- Set Line Data  
- Set Drive Data  

**Node Details:**  

- **Get Line File:**  
  - Type: HTTP Request  
  - Configuration: Downloads file content using LINE Messaging API URL constructed dynamically with message ID  
  - Authentication: HTTP header with LINE Channel Access Token (credentialed)  
  - Outputs: Binary file data of image/pdf  
  - Failures: Auth errors, file not found, network timeouts  

- **Download File (Drive):**  
  - Type: Google Drive node  
  - Operation: Download file by Drive file ID  
  - Outputs: Binary file data  
  - Failures: API errors, file access permissions, network issues  

- **Set Line Data:**  
  - Type: Set node  
  - Config: Adds fields `source = "line"`, `fileName` from event, and `fileId`  
  - Purpose: Metadata tagging for downstream processing  

- **Set Drive Data:**  
  - Type: Set node  
  - Config: Adds fields `source = "GDrive"`, dynamically sets fileName with timestamp and extension, and `fileId` from permissions array (possible edge case if permissions array structure changes)  

---

#### 2.3 Data Consolidation

**Overview:**  
Combines data from both LINE and Google Drive into a single data stream for unified processing.

**Nodes Involved:**  
- Merge Data Sources  

**Node Details:**  
- Type: Merge node (combine mode)  
- Config: Merge inputs by position, effectively stacking LINE and Drive file data together  
- Input Connections: Outputs from Set Line Data and Set Drive Data  
- Output: Unified data object with source tags and file binary for OCR  

---

#### 2.4 File Type Determination

**Overview:**  
Determines whether the input file is an image or a PDF to route to the appropriate OCR processing.

**Nodes Involved:**  
- Check File Type  

**Node Details:**  
- Type: If node  
- Condition: Checks binary MIME type with regex matching for `image/(jpg|jpeg|png)`  
- True branch: Image OCR workflow  
- False branch: PDF OCR workflow (fallback)  
- Failure cases: Missing MIME type, unsupported formats, binary data absent  

---

#### 2.5 OCR Processing with Mistral AI

**Overview:**  
Uses Mistral Document AI to perform OCR on image or PDF documents, converting binary data into parsed text.

**Nodes Involved:**  
- Extract text IMG  
- Extract text PDF  

**Node Details:**  

- **Extract text IMG:**  
  - Type: Mistral AI node  
  - Config: Document type set to `image_url`, binary property used is the file's binary data  
  - Output: Parsed text in markdown format, usually first page only  

- **Extract text PDF:**  
  - Type: Mistral AI node  
  - Config: Document type inferred as PDF, binary property `data`  
  - Output: Parsed text in markdown format  

- Failures: API key invalid, quota exceeded, file encoding errors, unrecognized file content  

---

#### 2.6 AI-based Information Extraction

**Overview:**  
Extracts structured invoice or letter fields from OCR text using OpenAI’s language model with a custom system prompt.

**Nodes Involved:**  
- Information Extractor  
- OpenAI Chat Model  

**Node Details:**  

- **Information Extractor:**  
  - Type: LangChain Information Extractor node  
  - Config: Receives markdown text from OCR, uses a system prompt instructing extraction of specific attributes (book_id, date, subject, to, etc.)  
  - Output: JSON object with extracted fields  
  - Failure modes: Malformed text input, model timeout, incomplete extraction  

- **OpenAI Chat Model:**  
  - Type: LangChain LM Chat Model (OpenAI GPT-4o-mini)  
  - Used as AI backend for Information Extractor node  
  - Credential: OpenAI API key  
  - Failures: Rate limits, authentication errors  

---

#### 2.7 Data Logging

**Overview:**  
Appends the extracted structured data as a new row into a predefined Google Sheets spreadsheet.

**Nodes Involved:**  
- Write to Google Sheet  

**Node Details:**  
- Type: Google Sheets node  
- Operation: Append row  
- Sheet: Sheet1 (gid=0) in spreadsheet ID `1asLDGXnPA4K55RfLDGRkgQzOpnKezDoVibWgnGaGUJ0`  
- Columns: book_id, date, subject, to, attach, detail, signed_by, signed_by_position, contact_phone, contact_email, download_url  
- Failures: Spreadsheet access denied, quota exceeded, field mismatches  

---

#### 2.8 Post-Processing Actions

**Overview:**  
Performs source-dependent post-processing: replies to LINE users or moves Google Drive files to an archive folder.

**Nodes Involved:**  
- Check Source for Reply (If)  
- Reply to LINE  
- Move file  

**Node Details:**  

- **Check Source for Reply:**  
  - Type: If node  
  - Condition: Checks if source field equals "line"  
  - True branch: Reply to LINE  
  - False branch: Move file in Drive  

- **Reply to LINE:**  
  - Type: HTTP Request  
  - Config: POST to LINE Messaging API reply endpoint with a JSON message confirming data saved, includes extracted details in Thai language  
  - Authentication: HTTP header with LINE Channel Access Token  
  - Failures: Token expiry, network errors  

- **Move file:**  
  - Type: Google Drive node  
  - Operation: Move file from original folder to archive root folder (specified Drive ID and folder ID)  
  - Failures: Permission denied, file locked  

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                          | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                  |
|---------------------|--------------------------------|----------------------------------------|-------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note         | Sticky Note                    | Overview and environment variables     |                               |                             | LINE Bot Letter OCR Workflow features and environment/credential requirements                               |
| LINE Webhook        | Webhook                       | Receive LINE POST message               |                               | Get Line File                | Receive data from LINE, extracts messageId for file download, path `/webhook-linebot-test`                   |
| Sticky Note1        | Sticky Note                    | LINE Webhook explanation                |                               |                             |                                                                                                              |
| Get Line File       | HTTP Request                  | Download file from LINE API             | LINE Webhook                  | Set Line Data               | Download file from LINE using messageId                                                                     |
| Sticky Note2        | Sticky Note                    | Download File from LINE process         |                               |                             | Download file from LINE, check message type, set metadata                                                    |
| Set Line Data       | Set                           | Set metadata for LINE files             | Get Line File                 | Merge Data Sources          | Adds source, fileName, fileId fields                                                                          |
| Google Drive Trigger| Google Drive Trigger          | Trigger on new file in Drive folder     |                               | Download File (Drive)       | Monitor new file creation in Google Drive folder                                                              |
| Sticky Note3        | Sticky Note                    | Google Drive Trigger explanation        |                               |                             | Watch folder `$env.GDRIVE_INVOICE_FOLDER_ID`, event fileCreated                                              |
| Download File (Drive)| Google Drive                  | Download new file from Drive             | Google Drive Trigger          | Set Drive Data              |                                                                                                              |
| Set Drive Data      | Set                           | Set metadata for Drive files             | Download File (Drive)         | Merge Data Sources          | Adds source, fileName, fileId fields                                                                          |
| Merge Data Sources  | Merge                         | Combine LINE and Drive file data         | Set Line Data, Set Drive Data | Check File Type             | Unified data object for downstream processing                                                                 |
| Sticky Note4        | Sticky Note                    | Merge Data Sources explanation           |                               |                             |                                                                                                              |
| Check File Type     | If                            | Determine file type for routing OCR      | Merge Data Sources            | Extract text IMG, Extract text PDF | Regex match on mimeType to choose OCR path                                                                |
| Sticky Note5        | Sticky Note                    | File Type Check explanation              |                               |                             |                                                                                                              |
| Extract text IMG    | Mistral AI                    | OCR for image files                       | Check File Type (true)        | Information Extractor       | OCR using Mistral AI, encode image to Base64                                                                |
| Sticky Note6        | Sticky Note                    | Mistral OCR Image explanation            |                               |                             |                                                                                                              |
| Extract text PDF    | Mistral AI                    | OCR for PDF files                         | Check File Type (false)       | Information Extractor       | OCR using Mistral AI, encode PDF to Base64                                                                  |
| Sticky Note7        | Sticky Note                    | Mistral OCR Document explanation         |                               |                             |                                                                                                              |
| Information Extractor| LangChain Information Extractor | Extract structured data from OCR text    | Extract text IMG / PDF        | Write to Google Sheet       | Uses OpenAI with system prompt to extract fields like book_id, date, subject, etc.                           |
| OpenAI Chat Model   | LangChain LM Chat OpenAI       | AI backend for Information Extractor     |                             | Information Extractor (ai_languageModel) |                                                                                                      |
| Sticky Note8        | Sticky Note                    | AI Validation Agent explanation          |                             |                             |                                                                                                              |
| Write to Google Sheet| Google Sheets                 | Append structured data to spreadsheet    | Information Extractor         | Check Source for Reply      | Append data row to Google Sheets with extracted fields                                                      |
| Sticky Note9        | Sticky Note                    | Google Sheets Output explanation         |                             |                             |                                                                                                              |
| Check Source for Reply| If                           | Decide post-processing by source         | Write to Google Sheet         | Reply to LINE, Move file    |                                                                                                              |
| Reply to LINE       | HTTP Request                  | Send confirmation reply to LINE user     | Check Source for Reply (true) |                             | Confirm invoice saved with extracted details                                                                |
| Sticky Note10       | Sticky Note                    | LINE Reply explanation                    |                             |                             |                                                                                                              |
| Move file           | Google Drive                  | Move processed Drive files to archive    | Check Source for Reply (false)|                             | Moves file to archive folder in Drive                                                                        |
| Sticky Note11       | Sticky Note                    | File Source Handling explanation          |                             |                             | Differentiates processing by file origin                                                                     |
| Sticky Note12       | Sticky Note                    | Google Drive Move Folder explanation      |                             |                             |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create LINE Webhook Node:**  
   - Type: Webhook (POST)  
   - Path: `webhook-linebot-test`  
   - No auth, accepts JSON from LINE Messaging API  

2. **Add HTTP Request Node "Get Line File":**  
   - URL: `https://api-data.line.me/v2/bot/message/{{ $json.body.events[0].message.id }}/content`  
   - Response format: File (binary)  
   - Authentication: HTTP Header Auth with LINE Channel Access Token credential  
   - Connect from LINE Webhook  

3. **Add Set Node "Set Line Data":**  
   - Fields:  
     - `source` = `"line"`  
     - `fileName` = `{{$json.body.events[0].message.fileName}}`  
     - `fileId` = `{{$json.body.events[0].message.id}}`  
   - Include all other fields  
   - Connect from Get Line File  

4. **Add Google Drive Trigger:**  
   - Event: `fileCreated` in specific folder  
   - Folder ID: Environment variable `GDRIVE_INVOICE_FOLDER_ID`  
   - Poll every minute  

5. **Add Google Drive Node "Download File (Drive)":**  
   - Operation: Download  
   - File ID: `{{$json.id}}` from trigger  
   - Connect from Google Drive Trigger  

6. **Add Set Node "Set Drive Data":**  
   - Fields:  
     - `source` = `"GDrive"`  
     - `fileName` = `slip_{{$now.format("ddLLyyyyHHmm")}}.{{$json.fileExtension}}`  
     - `fileId` = `{{$json.permissions[1].id}}` (verify permissions array exists)  
   - Connect from Download File (Drive)  

7. **Add Merge Node "Merge Data Sources":**  
   - Mode: Combine (merge by position)  
   - Connect from Set Line Data and Set Drive Data  

8. **Add If Node "Check File Type":**  
   - Condition: Regex match on `{{$binary.data.mimeType}}` against `^image/(jpg|jpeg|png)$`  
   - Connect from Merge Data Sources  

9. **Add Mistral AI Node "Extract text IMG":**  
   - Document Type: `image_url`  
   - Binary Property: `data`  
   - Connect from Check File Type (true branch)  

10. **Add Mistral AI Node "Extract text PDF":**  
    - Binary Property: `data`  
    - Connect from Check File Type (false branch)  

11. **Add LangChain Information Extractor Node:**  
    - Text input: `{{$json.pages[0].markdown}}` from OCR output  
    - System Prompt: Expert extraction, output raw JSON with specified fields  
    - Attributes: book_id, date, subject, to, attach, detail, signed_by, contact_phone, contact_email, download_url, signed_by_position  
    - Connect from both Extract text IMG and Extract text PDF  

12. **Add Google Sheets Node "Write to Google Sheet":**  
    - Operation: Append  
    - Sheet ID: `1asLDGXnPA4K55RfLDGRkgQzOpnKezDoVibWgnGaGUJ0`  
    - Sheet Name: Sheet1 (gid=0)  
    - Map columns to extracted output fields  
    - Connect from Information Extractor  

13. **Add If Node "Check Source for Reply":**  
    - Condition: Check if `source == "line"`  
    - Connect from Write to Google Sheet  

14. **Add HTTP Request Node "Reply to LINE":**  
    - URL: `https://api.line.me/v2/bot/message/reply`  
    - Method: POST  
    - Body: JSON with replyToken from webhook and message text including extracted data fields  
    - Headers: Content-Type application/json, Authorization with LINE access token (HTTP Header Auth credential)  
    - Connect from Check Source for Reply (true branch)  

15. **Add Google Drive Node "Move file":**  
    - Operation: Move file by ID from Google Drive Trigger  
    - Target folder: Archive folder root or specified folder ID  
    - Connect from Check Source for Reply (false branch)  

**Credentials Setup:**  
- LINE Channel Access Token stored in HTTP Header Auth credential for HTTP Requests  
- Google Drive OAuth2 credentials for Drive nodes  
- Google Sheets OAuth2 credentials for Google Sheets node  
- Mistral Cloud API key credential for Mistral nodes  
- OpenAI API key credential for LangChain nodes  

**Environment Variables Required:**  
- `LINE_CHANNEL_ACCESS_TOKEN` for LINE API access  
- `GDRIVE_INVOICE_FOLDER_ID` for Drive monitoring  
- `MISTRAL_API_KEY` for OCR service  
- `GSHEET_ID` for target spreadsheet  

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| LINE Bot Letter OCR Workflow handles incoming government letters via LINE or Drive, uses AI for OCR and info extraction, logs to Sheets | Main workflow purpose summary                                                                                  |
| Required environment variables and credentials include LINE tokens, GDrive folder ID, Mistral API key, and Google Sheets ID          | Environment setup                                                                                               |
| Uses Mistral Document AI for OCR and OpenAI GPT-4o-mini for information extraction                                                    | AI service details                                                                                              |
| Reply messages to LINE users confirm successful data capture with extracted letter details                                            | User feedback mechanism                                                                                         |
| Google Drive processed files are moved to an archive folder to maintain organization                                                  | Drive file management                                                                                           |
| Workflow uses n8n stable features (version specifics: Webhook v2, Google Sheets v4.4, Google Drive v3, Mistral v1, LangChain v1.2)   | Version compatibility notes                                                                                     |

---

This document comprehensively covers the workflow structure, node functions, configuration, and reproduction instructions for automation engineers and AI agents.  
It enables precise understanding and modification of the Thai Letter OCR automation with Mistral AI and OpenAI integration.