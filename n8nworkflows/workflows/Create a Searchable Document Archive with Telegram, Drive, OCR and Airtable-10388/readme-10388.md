Create a Searchable Document Archive with Telegram, Drive, OCR and Airtable

https://n8nworkflows.xyz/workflows/create-a-searchable-document-archive-with-telegram--drive--ocr-and-airtable-10388


# Create a Searchable Document Archive with Telegram, Drive, OCR and Airtable

### 1. Workflow Overview

This workflow, titled **Telegram Smart File Manager (Drive + OCR Integration)**, automates the process of receiving files and images from a Telegram bot, storing them in Google Drive, extracting searchable text using OCR for supported file types, indexing the metadata and extracted text in Airtable, and enabling search queries via Telegram commands. It also sends confirmation or error messages back to the user through Telegram.

**Target Use Cases:**  
- Users sending documents or images to a Telegram bot for automatic archiving.  
- Automatic OCR text extraction for image/PDF files to make documents searchable.  
- Storing files securely in Google Drive and indexing metadata & text in Airtable.  
- Searching stored documents by text content through Telegram commands.

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving and filtering Telegram messages to detect files/images or search commands.  
- **1.2 File Metadata Extraction & Download:** Extracting file details and downloading the file from Telegram.  
- **1.3 File Storage:** Uploading the downloaded file to Google Drive.  
- **1.4 OCR Processing:** Determining OCR eligibility, invoking Google Vision OCR, or skipping OCR.  
- **1.5 Indexing:** Merging OCR/non-OCR results and storing metadata and text in Airtable.  
- **1.6 User Notification:** Sending success or error messages to Telegram users.  
- **1.7 Search Handling:** Processing /search commands, querying Airtable, and returning results.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Listens for incoming Telegram messages and routes them based on content type: files/images get processed for storage; text messages starting with `/search` trigger a search flow.

**Nodes Involved:**  
- Telegram Trigger  
- Check if File/Image (IF node)  
- Check Search Command (IF node)  
- Sticky: Trigger  
- Sticky: Check File  
- Sticky: Search Cmd  

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger (Webhook listener)  
  - Config: Listens for `message` and `edited_message` updates from Telegram bot.  
  - Credentials: Telegram OAuth (configured)  
  - Inputs: Incoming Telegram updates  
  - Outputs: Two paths to "Check if File/Image" and "Check Search Command" nodes  
  - Failures: Telegram API downtime, webhook misconfiguration  
  - Notes: Entry point of entire workflow.

- **Check if File/Image**  
  - Type: IF node  
  - Config: Checks if incoming message contains either `document` or `photo` object.  
  - Conditions combined with OR: existence of `$json.message.document` or `$json.message.photo`.  
  - Outputs: True path for files/images, False path ignored (no downstream).  
  - Failures: Expression errors if message structure varies unexpectedly.

- **Check Search Command**  
  - Type: IF node  
  - Config: Verifies if message text starts with `/search ` command (case-sensitive).  
  - Outputs: True path to search extraction, False path ignored.  
  - Failures: None expected; edge if message is non-text without proper fallback.

- **Sticky Notes**  
  - Provide visual clarification for filtering files and detecting search commands.

---

#### 1.2 File Metadata Extraction & Download

**Overview:**  
Extracts essential metadata from Telegram message (file ID, name, MIME type, user and chat info). Then downloads the actual file content from Telegram servers.

**Nodes Involved:**  
- Extract File Metadata (Set node)  
- Download File from Telegram (Telegram node)  
- Sticky: Extract Metadata  
- Sticky: Download File  

**Node Details:**  

- **Extract File Metadata**  
  - Type: Set node  
  - Config: Assigns extracted values:  
    - `fileId`: from `document.file_id` or last photo's `file_id`  
    - `fileName`: document name or synthesized image name with timestamp  
    - `mimeType`: document MIME or default `image/jpeg`  
    - `chatId`: chat identifier  
    - `userName`: first and last name concatenated  
  - Outputs: Enriched JSON with metadata  
  - Failures: Missing fields may cause undefined values; handled with conditional chaining.

- **Download File from Telegram**  
  - Type: Telegram node  
  - Config: Downloads file binary using `fileId` from previous node.  
  - Credentials: Telegram OAuth  
  - Outputs: Binary data of the file  
  - Failures: API errors, invalid file ID, network issues.

- **Sticky Notes**  
  - Explain purpose of metadata extraction and file download.

---

#### 1.3 File Storage

**Overview:**  
Uploads the downloaded file to Google Drive into the root folder, returning the Drive file metadata.

**Nodes Involved:**  
- Upload to Google Drive (Google Drive node)  
- Sticky: Upload to Drive  

**Node Details:**  

- **Upload to Google Drive**  
  - Type: Google Drive node  
  - Config: Uploads file binary to root folder of Google Drive "My Drive"  
  - Credentials: OAuth2 for Google Drive  
  - Inputs: Binary file from Telegram Download node  
  - Outputs: Drive file metadata including webViewLink for sharing  
  - Failures: Invalid credentials, Drive API quota, file size limits.

- **Sticky Note**  
  - Highlights cloud upload step.

---

#### 1.4 OCR Processing

**Overview:**  
Determines if the uploaded file requires OCR (only images or PDFs). If eligible, invokes Google Vision OCR API to extract text; otherwise, sets default "no OCR" values.

**Nodes Involved:**  
- Check if OCR Eligible (IF node)  
- Google Vision OCR (HTTP Request node)  
- Extract OCR Text (Set node)  
- No OCR Needed (Set node)  
- Merge OCR Paths (Merge node)  
- Sticky: OCR Eligible, Vision OCR, Extract OCR Text, No OCR  

**Node Details:**  

- **Check if OCR Eligible**  
  - Type: IF node  
  - Conditions: MIME type starts with `image/` OR equals `application/pdf`  
  - True: Google Vision OCR path  
  - False: No OCR Needed path  
  - Failures: Unrecognized MIME, malformed data.

- **Google Vision OCR**  
  - Type: HTTP Request node  
  - Config: POST to Google Vision API `images:annotate` endpoint  
  - Auth: OAuth2 (Google)  
  - Body: JSON including base64-encoded image content from Telegram download binary  
  - Outputs: OCR JSON response  
  - Failures: API limits, invalid image data, auth errors.

- **Extract OCR Text**  
  - Type: Set node  
  - Parses OCR response to extract `fullTextAnnotation.text` or sets "No text detected"  
  - Assigns a confidence label `"High"` if text found, else `"None"`  
  - Failures: Unexpected response shape.

- **No OCR Needed**  
  - Type: Set node  
  - Sets default text `"Not applicable for this file type"` and confidence `"N/A"`  
  - Used when OCR not applicable.

- **Merge OCR Paths**  
  - Type: Merge node  
  - Combines the true OCR path and false no OCR path outputs into a single stream for indexing.

- **Sticky Notes**  
  - Clarify OCR eligibility check, OCR process, text extraction, and skipping OCR.

---

#### 1.5 Indexing

**Overview:**  
Stores the file metadata and extracted text content in an Airtable base for future searchability.

**Nodes Involved:**  
- Index in Airtable (Airtable node)  
- Sticky: Airtable Index  

**Node Details:**  

- **Index in Airtable**  
  - Type: Airtable node  
  - Operation: Create new record in specified base and table  
  - Inputs: Metadata (file name, Drive link, user info) and extracted OCR text from Merge node  
  - Credentials: Airtable API token  
  - Failures: API rate limits, invalid base/table IDs, data mapping errors.

- **Sticky Note**  
  - Explains purpose of metadata + OCR text indexing.

---

#### 1.6 User Notification

**Overview:**  
Sends a confirmation message with file details and preview on success or an error message on failure.

**Nodes Involved:**  
- Send Success Message (Telegram node)  
- Send Error Message (Telegram node)  
- Sticky: Success Msg, Error Msg  

**Node Details:**  

- **Send Success Message**  
  - Type: Telegram node  
  - Sends markdown-formatted message including file name, Google Drive link, snippet of extracted text, and timestamp  
  - Credentials: Telegram OAuth  
  - Inputs: Data from Airtable indexing node  
  - Failures: Telegram API errors, formatting issues.

- **Send Error Message**  
  - Type: Telegram node  
  - Sends a generic error message if processing fails at any point downstream from "No OCR Needed" node  
  - Inputs: Chat ID for user notification  
  - Failures: Telegram API errors.

- **Sticky Notes**  
  - Clarify notification message roles.

---

#### 1.7 Search Handling

**Overview:**  
Handles `/search` command from users, extracts the query, searches Airtable indexed text, and sends back matching file records with previews.

**Nodes Involved:**  
- Extract Search Query (Set node)  
- Search in Index (Airtable node)  
- Send Search Results (Telegram node)  
- Sticky: Extract Query, Search Index, Search Results  

**Node Details:**  

- **Extract Search Query**  
  - Type: Set node  
  - Extracts search keywords by removing `/search ` prefix and trims whitespace  
  - Stores chatId for reply  
  - Failures: Input text missing or malformed.

- **Search in Index**  
  - Type: Airtable node  
  - Operation: Search records where `extracted_text` field contains the query string (case insensitive)  
  - Uses Airtable formula with `SEARCH(LOWER("query"), LOWER({extracted_text}))`  
  - Credentials: Airtable API token  
  - Failures: API limits, formula syntax errors.

- **Send Search Results**  
  - Type: Telegram node  
  - Sends markdown message listing matching files with file names, Drive URLs, and text preview snippets (up to 100 chars)  
  - If no results, sends "No results found."  
  - Credentials: Telegram OAuth  
  - Failures: Telegram API errors, formatting.

- **Sticky Notes**  
  - Explain detection, extraction, search logic, and result sending.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                                | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                   |
|----------------------------|------------------------|-----------------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger       | Entry point: listens for Telegram messages    | -                            | Check if File/Image, Check Search Command | 🎯 **Trigger**: Listens for new messages/documents from users                                |
| Check if File/Image         | IF                     | Filters messages to files/images               | Telegram Trigger             | Extract File Metadata        | 🔍 **Filter**: Only proceed if message has file or image                                    |
| Sticky: Check File          | Sticky Note            | Clarifies filtering files/images               | -                            | -                           | 🔍 **Filter**: Only proceed if message has file or image                                    |
| Extract File Metadata       | Set                    | Extracts file ID, name, MIME, user/chat info  | Check if File/Image          | Download File from Telegram  | 📋 **Extract**: fileId, name, MIME, chat & user info                                        |
| Sticky: Extract Metadata    | Sticky Note            | Clarifies metadata extraction                   | -                            | -                           | 📋 **Extract**: fileId, name, MIME, chat & user info                                        |
| Download File from Telegram | Telegram node          | Downloads file binary content                   | Extract File Metadata        | Upload to Google Drive       | ⬇️ **Download**: Get file binary from Telegram API                                          |
| Sticky: Download File       | Sticky Note            | Clarifies download step                         | -                            | -                           | ⬇️ **Download**: Get file binary from Telegram API                                          |
| Upload to Google Drive      | Google Drive           | Uploads file to Google Drive root folder       | Download File from Telegram  | Check if OCR Eligible        | ☁️ **Upload**: Save file to Google Drive root                                               |
| Sticky: Upload to Drive     | Sticky Note            | Clarifies upload to Google Drive                | -                            | -                           | ☁️ **Upload**: Save file to Google Drive root                                               |
| Check if OCR Eligible       | IF                     | Checks if file is image/PDF for OCR             | Upload to Google Drive       | Google Vision OCR, No OCR Needed | 🧠 **OCR Check**: Is it image or PDF?                                                      |
| Sticky: OCR Eligible        | Sticky Note            | Clarifies OCR eligibility check                 | -                            | -                           | 🧠 **OCR Check**: Is it image or PDF?                                                      |
| Google Vision OCR           | HTTP Request           | Calls Google Vision API for OCR text extraction | Check if OCR Eligible (true) | Extract OCR Text             | 🔍 **OCR**: Extract text via Google Vision API                                              |
| Sticky: Vision OCR          | Sticky Note            | Clarifies OCR API call                           | -                            | -                           | 🔍 **OCR**: Extract text via Google Vision API                                              |
| Extract OCR Text            | Set                    | Parses OCR response to extract text             | Google Vision OCR            | Merge OCR Paths             | ✂️ **Parse**: Get fullTextAnnotation from OCR response                                     |
| Sticky: Extract OCR Text    | Sticky Note            | Clarifies OCR text extraction                    | -                            | -                           | ✂️ **Parse**: Get fullTextAnnotation from OCR response                                     |
| No OCR Needed              | Set                    | Sets default text for non-OCR files             | Check if OCR Eligible (false)| Merge OCR Paths             | ➖ **Skip OCR**: Non-image/PDF files                                                        |
| Sticky: No OCR             | Sticky Note            | Clarifies skipping OCR                           | -                            | -                           | ➖ **Skip OCR**: Non-image/PDF files                                                        |
| Merge OCR Paths            | Merge                  | Merges OCR and non-OCR results                   | Extract OCR Text, No OCR Needed | Index in Airtable          |                                                                                              |
| Index in Airtable          | Airtable                | Stores metadata and extracted text               | Merge OCR Paths             | Send Success Message         | 📊 **Index**: Save metadata + OCR text to Airtable                                         |
| Sticky: Airtable Index     | Sticky Note             | Clarifies Airtable indexing                      | -                            | -                           | 📊 **Index**: Save metadata + OCR text to Airtable                                         |
| Send Success Message       | Telegram node           | Sends confirmation message to user               | Index in Airtable           | -                           | ✅ **Notify**: Success message with link & preview                                         |
| Sticky: Success Msg        | Sticky Note             | Clarifies success notification                    | -                            | -                           | ✅ **Notify**: Success message with link & preview                                         |
| Send Error Message         | Telegram node           | Sends error message on processing failure        | No OCR Needed (error path)  | -                           | ❌ **Error**: Notify user on failure                                                       |
| Sticky: Error Msg          | Sticky Note             | Clarifies error notification                      | -                            | -                           | ❌ **Error**: Notify user on failure                                                       |
| Check Search Command       | IF                      | Detects `/search` command in message text         | Telegram Trigger            | Extract Search Query         | 🔎 **Detect**: /search <query> command                                                    |
| Sticky: Search Cmd         | Sticky Note             | Clarifies search command detection                 | -                            | -                           | 🔎 **Detect**: /search <query> command                                                    |
| Extract Search Query       | Set                     | Extracts keywords from `/search` command           | Check Search Command        | Search in Index              | ✂️ **Parse**: Extract query after /search                                                 |
| Sticky: Extract Query      | Sticky Note             | Clarifies query extraction                          | -                            | -                           | ✂️ **Parse**: Extract query after /search                                                 |
| Search in Index            | Airtable                 | Searches Airtable for matching extracted text       | Extract Search Query        | Send Search Results          | 🔍 **Search**: Airtable full-text in extracted_text                                      |
| Sticky: Search Index       | Sticky Note             | Clarifies Airtable search                            | -                            | -                           | 🔍 **Search**: Airtable full-text in extracted_text                                      |
| Send Search Results        | Telegram node            | Sends search results with file links and preview    | Search in Index            | -                           | 📋 **Results**: Send matched files with previews                                         |
| Sticky: Search Results     | Sticky Note             | Clarifies sending of search results                  | -                            | -                           | 📋 **Results**: Send matched files with previews                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook to listen for `message` and `edited_message` updates.  
   - Set credentials with your Telegram Bot API token.

2. **Add IF Node "Check if File/Image"**  
   - Condition:  
     - `$json.message.document` exists OR  
     - `$json.message.photo` exists  
   - Connect Telegram Trigger → Check if File/Image (True path).

3. **Add Sticky Note "Check File"** near IF node to denote filtering.

4. **Add Set Node "Extract File Metadata"**  
   - Extract and assign:  
     - `fileId` := `$json.message.document.file_id` or last photo `file_id`  
     - `fileName` := `$json.message.document.file_name` or `"image_" + $json.message.date + ".jpg"`  
     - `mimeType` := `$json.message.document.mime_type` or `"image/jpeg"`  
     - `chatId` := `$json.message.chat.id`  
     - `userName` := `$json.message.from.first_name + " " + ($json.message.from.last_name || "")`  
   - Connect Check if File/Image (True) → Extract File Metadata.

5. **Add Telegram Node "Download File from Telegram"**  
   - Resource: File  
   - File ID: Use expression from `fileId` in metadata node.  
   - Use Telegram credentials.  
   - Connect Extract File Metadata → Download File from Telegram.

6. **Add Sticky Note "Extract Metadata"** near metadata extraction node.

7. **Add Sticky Note "Download File"** near download node.

8. **Add Google Drive Node "Upload to Google Drive"**  
   - Upload file binary from Telegram download node.  
   - Destination folder: root folder of "My Drive".  
   - Use Google Drive OAuth2 credentials.  
   - Connect Download File from Telegram → Upload to Google Drive.

9. **Add Sticky Note "Upload to Drive"** near upload node.

10. **Add IF Node "Check if OCR Eligible"**  
    - Condition:  
      - `mimeType` starts with `"image/"` OR  
      - `mimeType` equals `"application/pdf"`  
    - Connect Upload to Google Drive → Check if OCR Eligible.

11. **Add Sticky Note "OCR Eligible"** near OCR check node.

12. **Add HTTP Request Node "Google Vision OCR"**  
    - Method: POST  
    - URL: `https://vision.googleapis.com/v1/images:annotate`  
    - Authentication: OAuth2 (Google)  
    - Body: JSON with image content (base64-encoded binary from Telegram download node) and feature type TEXT_DETECTION.  
    - Connect Check if OCR Eligible (True) → Google Vision OCR.

13. **Add Sticky Note "Vision OCR"** near OCR node.

14. **Add Set Node "Extract OCR Text"**  
    - Extract `fullTextAnnotation.text` from OCR response or set "No text detected".  
    - Assign confidence `"High"` or `"None"`.  
    - Connect Google Vision OCR → Extract OCR Text.

15. **Add Sticky Note "Extract OCR Text"** near text extraction node.

16. **Add Set Node "No OCR Needed"**  
    - Set `extractedText` to `"Not applicable for this file type"`.  
    - Set `confidence` to `"N/A"`.  
    - Connect Check if OCR Eligible (False) → No OCR Needed.

17. **Add Sticky Note "No OCR"** near no OCR node.

18. **Add Merge Node "Merge OCR Paths"**  
    - Merge mode: Wait for all inputs (default).  
    - Connect Extract OCR Text → Merge OCR Paths (main input 0)  
    - Connect No OCR Needed → Merge OCR Paths (main input 1).

19. **Add Airtable Node "Index in Airtable"**  
    - Operation: Create record  
    - Base and Table: Use your Airtable IDs.  
    - Map fields for file metadata, Drive file URL, extracted OCR text, user info, etc.  
    - Use Airtable API token credentials.  
    - Connect Merge OCR Paths → Index in Airtable.

20. **Add Sticky Note "Airtable Index"** near Airtable node.

21. **Add Telegram Node "Send Success Message"**  
    - Text: Markdown with file name, Drive link, snippet of extracted text (up to 200 chars), and timestamp.  
    - Chat ID from metadata.  
    - Credentials: Telegram OAuth.  
    - Connect Index in Airtable → Send Success Message.

22. **Add Sticky Note "Success Msg"** near success message node.

23. **Add Telegram Node "Send Error Message"**  
    - Message: Generic error text.  
    - Chat ID from metadata.  
    - Credentials: Telegram OAuth.  
    - Connect No OCR Needed node (error output) → Send Error Message.

24. **Add Sticky Note "Error Msg"** near error message node.

25. **Add IF Node "Check Search Command"**  
    - Condition: Text starts with `/search `  
    - Connect Telegram Trigger → Check Search Command.

26. **Add Sticky Note "Search Cmd"** near search command node.

27. **Add Set Node "Extract Search Query"**  
    - Extract `searchQuery` by removing `/search ` prefix and trimming.  
    - Assign `chatId` for reply.  
    - Connect Check Search Command (True) → Extract Search Query.

28. **Add Sticky Note "Extract Query"** near query extraction node.

29. **Add Airtable Node "Search in Index"**  
    - Operation: Search records in Airtable base/table where `extracted_text` contains `searchQuery` (case insensitive).  
    - Use formula: `SEARCH(LOWER("searchQuery"), LOWER({extracted_text}))`  
    - Credentials: Airtable API token.  
    - Connect Extract Search Query → Search in Index.

30. **Add Sticky Note "Search Index"** near search node.

31. **Add Telegram Node "Send Search Results"**  
    - Send back a markdown list of matched files with file name, Drive link, and preview snippet (up to 100 chars).  
    - If none found, message "No results found."  
    - Chat ID from `chatId`.  
    - Credentials: Telegram OAuth.  
    - Connect Search in Index → Send Search Results.

32. **Add Sticky Note "Search Results"** near results node.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow integrates Telegram, Google Drive, Google Vision OCR, and Airtable to create a searchable document archive. | This multi-service integration allows automated file handling, OCR text extraction, and indexing.|
| The Google Vision API requires OAuth2 credentials with enabled Vision API access.                                        | https://cloud.google.com/vision/docs/setup                                                        |
| Airtable base and table IDs must be configured correctly with appropriate API token credentials.                        | https://airtable.com/api                                                                          |
| Telegram bot must have webhook URL configured and accessible for the Telegram Trigger node to receive updates.           | https://core.telegram.org/bots/api#webhooks                                                     |
| The workflow assumes images and PDFs only for OCR processing; other file types skip OCR.                                  | Consider extending MIME type checks for additional file types if needed.                        |
| Telegram message formatting uses Markdown; ensure special characters are escaped in text outputs.                        | https://core.telegram.org/bots/api#markdown-style                                                |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. All processing respects applicable content policies and contains no illegal, offensive, or protected material. Only legal and public data is handled.