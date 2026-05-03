Parse PDF, DOCX & Images with Mistral OCR via Google Drive with Slack Alerts

https://n8nworkflows.xyz/workflows/parse-pdf--docx---images-with-mistral-ocr-via-google-drive-with-slack-alerts-9084


# Parse PDF, DOCX & Images with Mistral OCR via Google Drive with Slack Alerts

### 1. Workflow Overview

This workflow automates OCR processing of PDF, DOCX, and image files uploaded or updated in a monitored Google Drive folder. It targets use cases where documents need to be parsed and converted into text and markdown for further processing, such as content ingestion, validation, or indexing. The workflow extracts text and images using Mistral OCR, organizes outputs into Google Drive folders, logs progress in Google Sheets, and sends Slack notifications on success or errors.

The main logical blocks are:

- **1.1 Input Reception:** Monitor Google Drive for new or updated files.
- **1.2 File Organization:** Create destination folders and copy source files.
- **1.3 File Type Determination:** Identify file type to select appropriate OCR processing path.
- **1.4 OCR Processing:** Download files, send to Mistral OCR API and retrieve parsed data.
- **1.5 Output Handling:** Save raw OCR JSON, per-page Markdown, aggregate Markdown, and extracted images to Google Drive.
- **1.6 Progress Tracking:** Log processing status and metadata in Google Sheets.
- **1.7 Notifications:** Send Slack alerts on success or unsupported file types.
- **1.8 Configuration & Credentials:** Manage workflow configuration variables and credential setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Detects new or updated files in a specified Google Drive folder every 5 minutes, initiating processing.

**Nodes Involved:**  
- File Created  
- File Updated  
- Loop over Items

**Node Details:**

- **File Created**  
  - Type: Google Drive Trigger  
  - Watches for new files created in a specific folder (folder ID must be set in parameters).  
  - Poll interval: every 5 minutes.  
  - Outputs metadata of new files.  
  - Potential failures: Google authentication errors, API rate limits, missing folder ID.

- **File Updated**  
  - Type: Google Drive Trigger  
  - Watches for files updated in the same folder every 5 minutes.  
  - Same failure modes as File Created.

- **Loop over Items**  
  - Type: SplitInBatches  
  - Splits multiple trigger events into single item processing for stable sequential handling.  
  - Passes each file metadata item downstream.

---

#### 2.2 File Organization

**Overview:**  
Prepares the destination folder structure for processed files by creating a uniquely named folder and copying the source file into it.

**Nodes Involved:**  
- Set File ID  
- Workflow Configuration  
- Create Folder  
- Create Entry in sheet  
- Copy file

**Node Details:**

- **Set File ID**  
  - Type: Set  
  - Extracts and assigns key file metadata: file ID, name, destination folder name (based on filename without extension), MIME type.  
  - Used for downstream referencing.

- **Workflow Configuration**  
  - Type: Set  
  - Stores configuration variables: Google Sheet ID and destination folder ID (root folder for output).  
  - Must be manually configured before running.

- **Create Folder**  
  - Type: Google Drive  
  - Creates a new folder named `<original filename>_<timestamp>` inside the configured destination folder.  
  - Stores the new folder ID for saving outputs.

- **Create Entry in sheet**  
  - Type: Google Sheets  
  - Adds a new row to the tracking sheet with status "Started", start timestamp, directory URL, and file name.  
  - Enables progress tracking.

- **Copy file**  
  - Type: Google Drive  
  - Copies the original file into the newly created output folder.  
  - Ensures original file remains untouched and output folder contains the input copy.

**Potential Failures:**  
- Folder creation failure due to permissions or quota.  
- Google Sheets API errors (invalid sheet ID, insufficient permissions).  
- Copy file permission issues.

---

#### 2.3 File Type Determination

**Overview:**  
Determines whether the file is a document (PDF/DOCX), an image (PNG/JPEG), or unsupported, to route processing accordingly.

**Nodes Involved:**  
- Add Source File in Sheet  
- Determine Document Type  
- Use Document Url  
- Use Image Url  
- Set Error in Sheet  
- Send Error Message

**Node Details:**

- **Add Source File in Sheet**  
  - Type: Google Sheets  
  - Updates sheet with source file link and output directory link for tracking.

- **Determine Document Type**  
  - Type: Switch  
  - Uses regex to check MIME type for document or image types:  
    - Documents: `application/pdf`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`  
    - Images: `image/png`, `image/jpeg`, `image/jpg`  
  - Routes accordingly: document, image, or fallback (unsupported).

- **Use Document Url**  
  - Type: Set  
  - Sets the variable `document_type` to `"document_url"` for Mistral OCR API call.

- **Use Image Url**  
  - Type: Set  
  - Sets the variable `document_type` to `"image_url"` for Mistral OCR API call.

- **Set Error in Sheet**  
  - Type: Google Sheets  
  - Marks the Job as "Unsupported Type" with completion timestamp for unsupported MIME types.

- **Send Error Message**  
  - Type: Slack  
  - Sends notification to Slack channel about unsupported file type with file link and output folder link.

**Potential Failures:**  
- Incorrect or missing MIME type leading to wrong routing.  
- Unsupported file types not handled gracefully.  
- Slack API authentication or channel permission failures.

---

#### 2.4 OCR Processing

**Overview:**  
Downloads the copied file from Google Drive and sends it to Mistral OCR API for parsing.

**Nodes Involved:**  
- Use Url  
- Download File  
- OCR Document

**Node Details:**

- **Use Url**  
  - Type: Set  
  - Passes the previously set `document_type` variable downstream to form the API request.

- **Download File**  
  - Type: Google Drive  
  - Downloads the copied file binary content for sending to OCR.  
  - Outputs binary data of the file.

- **OCR Document**  
  - Type: HTTP Request  
  - Calls Mistral OCR API endpoint using POST.  
  - Sends JSON body containing:  
    - Model name: `"mistral-ocr-latest"`  
    - Document type and file content encoded as Base64 with MIME type header.  
    - Request includes `include_image_base64` flag for images.  
  - Uses Mistral Cloud API credentials for authentication.  
  - Receives parsed OCR JSON including pages and images.

**Potential Failures:**  
- Download failures due to file permission issues or network errors.  
- Mistral API authentication errors or quota limits.  
- Timeout or malformed responses from OCR API.  
- Large files causing memory or timeout issues.

---

#### 2.5 Output Handling

**Overview:**  
Processes Mistral OCR output, saving raw JSON, multiple Markdown files per page, aggregated Markdown, and extracted images into Google Drive folders.

**Nodes Involved:**  
- Save raw OCR file  
- Split Pages  
- Split Images  
- Extract Base64 value  
- Convert to File  
- Save Image  
- Save Page MD File  
- Aggregate  
- Code in JavaScript  
- Save MD File  
- Add MD file in sheet  
- Add json file in sheet  
- Merge  
- Merge1

**Node Details:**

- **Save raw OCR file**  
  - Type: Google Drive  
  - Saves the complete OCR JSON output as `<filename>.json` in the output folder.

- **Split Pages**  
  - Type: SplitOut  
  - Splits the OCR JSON pages array into individual items for per-page processing.

- **Split Images**  
  - Type: SplitOut  
  - Extracts images array from each page for processing.

- **Extract Base64 value**  
  - Type: Set  
  - Strips MIME type header from Base64 image string, retaining raw Base64 data.

- **Convert to File**  
  - Type: ConvertToFile  
  - Converts Base64 string into binary file with correct MIME type for saving.

- **Save Image**  
  - Type: Google Drive  
  - Saves binary image files into the output folder.

- **Save Page MD File**  
  - Type: Google Drive  
  - Saves a Markdown file for each page named `<filename>_page_XX.md` with OCR text content.

- **Aggregate**  
  - Type: Aggregate  
  - Aggregates all page Markdown data into a single array.

- **Code in JavaScript**  
  - Type: Code  
  - Concatenates all page Markdown into one aggregated Markdown string.

- **Save MD File**  
  - Type: Google Drive  
  - Saves the aggregated Markdown as `<filename>.md` in the output folder.

- **Add MD file in sheet**  
  - Type: Google Sheets  
  - Adds or updates the sheet with link to the aggregated Markdown file.

- **Add json file in sheet**  
  - Type: Google Sheets  
  - Adds or updates the sheet with link to the raw OCR JSON file.

- **Merge** & **Merge1**  
  - Type: Merge  
  - Used to coordinate multiple asynchronous branches and consolidate data streams before final steps.

**Potential Failures:**  
- File save errors due to Google Drive permission or quota.  
- Binary conversion failures if Base64 is malformed.  
- Large files causing memory or timeout problems.  
- Sheet update errors.

---

#### 2.6 Progress Tracking

**Overview:**  
Updates the Google Sheet with processing status, timestamps, and links for each file.

**Nodes Involved:**  
- Create Entry in sheet  
- Add Source File in Sheet  
- Add MD file in sheet  
- Add json file in sheet  
- Set Error in Sheet  
- Complete job in sheet

**Node Details:**

- Each Google Sheets node appends or updates rows with relevant metadata: file name, links, directory URL, status, start and completion timestamps.

- Statuses include `Started`, `Done`, and `Unsupported Type`.

- Uses `Directory` column as matching key for updates.

**Potential Failures:**  
- Sheet not found or permission denied errors.  
- Concurrent writes may cause race conditions.

---

#### 2.7 Notifications

**Overview:**  
Send Slack notifications about success or failure (unsupported type) for each processed file.

**Nodes Involved:**  
- Send Success Message  
- Send Error Message

**Node Details:**

- Both nodes use Slack OAuth2 credentials.

- Messages include links to the processed file and output directory.

- Success message triggered after job completion.

- Error message triggered if unsupported file type detected.

**Potential Failures:**  
- Slack authentication failures.  
- Channel permission errors.  
- Network issues preventing message delivery.

---

#### 2.8 Configuration & Credentials

**Overview:**  
Manages setup variables and credentials required by the workflow.

**Nodes Involved:**  
- Workflow Configuration  
- Sticky Note nodes (documentation and instructions)

**Node Details:**

- **Workflow Configuration** node defines:  
  - `google_sheet_id` (Google Sheets document ID)  
  - `dest_folder_id` (Google Drive folder ID for outputs)

- Sticky Notes provide detailed instructions and links for:  
  - Google Drive and Sheets credentials setup  
  - Mistral Cloud API credentials  
  - Slack OAuth2 credentials  
  - How to configure trigger folder IDs and sheet IDs  
  - Workflow overview and usage guidance

---

### 3. Summary Table

| Node Name            | Node Type               | Functional Role                             | Input Node(s)                 | Output Node(s)                        | Sticky Note |
|----------------------|-------------------------|--------------------------------------------|------------------------------|-------------------------------------|-------------|
| File Created         | Google Drive Trigger    | Trigger on new files in folder              | —                            | Loop over Items                     | See Sticky Note "Input Reception" |
| File Updated         | Google Drive Trigger    | Trigger on updated files in folder          | —                            | Loop over Items                     | See Sticky Note "Input Reception" |
| Loop over Items      | SplitInBatches          | Process each file item individually          | File Created, File Updated    | Set File ID                        |             |
| Set File ID          | Set                     | Extract key file info                         | Loop over Items               | Workflow Configuration             |             |
| Workflow Configuration| Set                    | Store config variables                        | Set File ID                  | Create Folder                     | See Sticky Note "Configuration"    |
| Create Folder        | Google Drive            | Create destination folder                     | Workflow Configuration       | Create Entry in sheet              | See Sticky Note "File Organization"|
| Create Entry in sheet| Google Sheets           | Log job start in sheet                        | Create Folder                | Copy file                        |             |
| Copy file            | Google Drive            | Copy source file to destination folder       | Create Entry in sheet        | Add Source File in Sheet           |             |
| Add Source File in Sheet| Google Sheets         | Log source file info                          | Copy file                    | Determine Document Type            |             |
| Determine Document Type| Switch                 | Route based on MIME type                      | Add Source File in Sheet     | Use Document Url, Use Image Url, Set Error in Sheet |             |
| Use Document Url     | Set                     | Set document type to 'document_url'          | Determine Document Type      | Use Url                         |             |
| Use Image Url        | Set                     | Set document type to 'image_url'             | Determine Document Type      | Use Url                         |             |
| Set Error in Sheet   | Google Sheets           | Log unsupported file type error               | Determine Document Type      | Send Error Message               |             |
| Send Error Message   | Slack                   | Notify unsupported file type                   | Set Error in Sheet           | Loop over Items (for batch)       | See Sticky Note "Notifications"      |
| Use Url              | Set                     | Pass document_type for OCR API call           | Use Document Url, Use Image Url | Download File               |             |
| Download File        | Google Drive            | Download copied file binary                    | Use Url                     | OCR Document                    |             |
| OCR Document         | HTTP Request            | Call Mistral OCR API                           | Download File               | Save raw OCR file, Split Pages    | See Sticky Note "OCR Processing"     |
| Save raw OCR file    | Google Drive            | Save full OCR JSON output                       | OCR Document                | Add json file in sheet            |             |
| Split Pages          | SplitOut                | Split OCR pages array                           | OCR Document                | Split Images, Save Page MD File, Aggregate |             |
| Split Images         | SplitOut                | Split images array per page                      | Split Pages                 | Extract Base64 value             |             |
| Extract Base64 value | Set                     | Strip MIME header from Base64 image             | Split Images                | Convert to File                  |             |
| Convert to File      | ConvertToFile           | Convert Base64 string to binary file             | Extract Base64 value        | Save Image                      |             |
| Save Image           | Google Drive            | Save extracted images as files                   | Convert to File             | Merge                          |             |
| Save Page MD File    | Google Drive            | Save per-page Markdown files                      | Split Pages                 | Merge                          |             |
| Aggregate            | Aggregate               | Aggregate page markdown into array                 | Split Pages                 | Code in JavaScript              |             |
| Code in JavaScript   | Code                    | Concatenate markdown pages into single markdown    | Aggregate                   | Save MD File                   |             |
| Save MD File         | Google Drive            | Save aggregated markdown file                       | Code in JavaScript           | Add MD file in sheet            |             |
| Add MD file in sheet | Google Sheets           | Log aggregated markdown file                         | Save MD File                | Merge                         |             |
| Add json file in sheet| Google Sheets           | Log raw OCR JSON file                               | Save raw OCR file           | Merge1                        |             |
| Merge                | Merge                   | Combine branches (images, MD files, others)          | Save Image, Save Page MD File, Add MD file in sheet | Merge1 |             |
| Merge1               | Merge                   | Final merge before completion                         | Add json file in sheet, Merge | Complete job in sheet          |             |
| Complete job in sheet| Google Sheets           | Update sheet marking job as 'Done'                   | Merge1                      | Send Success Message           |             |
| Send Success Message | Slack                   | Notify successful processing                          | Complete job in sheet       | Loop over Items (batch)          | See Sticky Note "Notifications"      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Triggers:**  
   - Add two Google Drive Trigger nodes named `File Created` and `File Updated`.  
   - Configure each to trigger on file creation or update respectively.  
   - Set the folder to watch by folder ID.  
   - Poll every 5 minutes.  
   - Assign Google Drive OAuth2 credentials.

2. **Add SplitInBatches Node:**  
   - Add a `Loop over Items` SplitInBatches node connected to both triggers.  
   - This processes files one by one.

3. **Set File Metadata:**  
   - Add a `Set File ID` node.  
   - Assign variables:  
     - `file_id` = `{{$json.id}}`  
     - `file_name` = `{{$json.name}}`  
     - `dest_folder` = filename without extension  
     - `file_name_no_ext` = filename without extension  
     - `mime_type` = `{{$json.mimeType}}`

4. **Store Workflow Configuration:**  
   - Add `Workflow Configuration` Set node.  
   - Define:  
     - `google_sheet_id` = your Google Sheet ID  
     - `dest_folder_id` = your destination Google Drive folder ID  
   - No direct connections; connect from `Set File ID`.

5. **Create Output Folder:**  
   - Add `Create Folder` Google Drive node.  
   - Name: `{{$json.dest_folder}}_{{$now.format('yyyyMMdd-hhmmss')}}`  
   - Parent folder ID: `{{$json.dest_folder_id}}` from config node.  
   - Use Google Drive credentials.

6. **Log Start in Google Sheets:**  
   - Add `Create Entry in sheet` Google Sheets node.  
   - Append row with status `Started`, start timestamp, directory URL, and file name.  
   - Use your Google Sheets credentials.  
   - Connect from `Create Folder`.

7. **Copy Source File:**  
   - Add `Copy file` Google Drive node.  
   - Copy original file (`file_id`) into newly created folder (`Create Folder` output ID).  
   - Use Google Drive credentials.  
   - Connect from `Create Entry`.

8. **Log Source File Info:**  
   - Add `Add Source File in Sheet` Google Sheets node.  
   - Append or update row with file URL and directory URL.  
   - Connect from `Copy file`.

9. **Determine MIME Type:**  
   - Add `Determine Document Type` Switch node.  
   - Conditions:  
     - Documents (PDF, DOCX) regex on `mime_type`  
     - Images (PNG, JPEG) regex on `mime_type`  
     - Fallback to unsupported.  
   - Connect from `Add Source File in Sheet`.

10. **Route Document Type:**  
    - For documents, connect to `Use Document Url` Set node setting `document_type` to `"document_url"`.  
    - For images, connect to `Use Image Url` Set node setting `document_type` to `"image_url"`.  
    - For unsupported, connect to `Set Error in Sheet`.

11. **Handle Unsupported File Type:**  
    - `Set Error in Sheet` Google Sheets node updates status to "Unsupported Type" and completion timestamp.  
    - Connect to `Send Error Message` Slack node.  
    - Slack node sends message with file and folder links.  
    - Use Slack OAuth2 credentials.

12. **Prepare for OCR:**  
    - Connect `Use Document Url` and `Use Image Url` to `Use Url` Set node to pass `document_type`.  
    - Connect `Use Url` to `Download File` Google Drive node to download the copied file binary.

13. **Call Mistral OCR API:**  
    - Add `OCR Document` HTTP Request node.  
    - POST to `https://api.mistral.ai/v1/ocr`.  
    - JSON body includes:  
      - Model: `"mistral-ocr-latest"`  
      - Document type and base64 encoded file with MIME type header.  
      - `include_image_base64` true.  
    - Use Mistral Cloud API credentials.  
    - Connect from `Download File`.

14. **Save Raw OCR JSON:**  
    - Add `Save raw OCR file` Google Drive node.  
    - Save JSON content of OCR result as `<filename>.json` in output folder.  
    - Connect from `OCR Document`.

15. **Split OCR Pages and Images:**  
    - Add `Split Pages` node splitting `body.pages` array.  
    - Connect from `OCR Document`.  
    - Add `Split Images` node splitting `images` array from each page.  
    - Connect from `Split Pages`.

16. **Process Images:**  
    - Add `Extract Base64 value` Set node to strip MIME header from image base64 string.  
    - Connect from `Split Images`.  
    - Add `Convert to File` node to convert base64 to binary file.  
    - Connect from `Extract Base64 value`.  
    - Add `Save Image` Google Drive node to save images in output folder.  
    - Connect from `Convert to File`.

17. **Process Markdown Pages:**  
    - Add `Save Page MD File` Google Drive node.  
    - Save per-page markdown files named `<filename>_page_XX.md`.  
    - Connect from `Split Pages`.

18. **Aggregate Markdown:**  
    - Add `Aggregate` node to collect page markdown fields.  
    - Connect from `Split Pages`.  
    - Add `Code in JavaScript` node to concatenate markdown array into single markdown string.  
    - Connect from `Aggregate`.

19. **Save Aggregated Markdown:**  
    - Add `Save MD File` Google Drive node.  
    - Save aggregated markdown as `<filename>.md`.  
    - Connect from `Code in JavaScript`.

20. **Update Google Sheets with Output Files:**  
    - Add `Add MD file in sheet` Google Sheets node for markdown link.  
    - Connect from `Save MD File`.  
    - Add `Add json file in sheet` Google Sheets node for OCR JSON link.  
    - Connect from `Save raw OCR file`.

21. **Merge Output Branches:**  
    - Add `Merge` node to combine `Save Image`, `Save Page MD File`, and `Add MD file in sheet` outputs.  
    - Connect from respective nodes.  
    - Add `Merge1` node to combine `Add json file in sheet` and `Merge`.  
    - Connect accordingly.

22. **Complete Job in Sheet:**  
    - Add `Complete job in sheet` Google Sheets node.  
    - Update status to `Done` and completion timestamp.  
    - Connect from `Merge1`.

23. **Send Success Slack Notification:**  
    - Add `Send Success Message` Slack node.  
    - Send notification with file and folder links.  
    - Connect from `Complete job in sheet`.

24. **Loop back to process next item:**  
    - Connect success and error Slack nodes back to `Loop over Items` to continue batch processing.

25. **Configure all nodes requiring credentials:**  
    - Assign Google Drive OAuth2 credentials to all Google Drive nodes.  
    - Assign Google Sheets OAuth2 credentials to all Google Sheets nodes.  
    - Assign Mistral Cloud API credentials to `OCR Document` node.  
    - Assign Slack OAuth2 credentials to both Slack nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Google Drive Trigger node docs: https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googledrivetrigger/ | For setting up the File Created and File Updated triggers |
| Google Credentials docs: https://docs.n8n.io/integrations/builtin/credentials/google/ | Necessary for Drive and Sheets nodes |
| Mistral Document OCR API docs: https://docs.mistral.ai/capabilities/document_ai/basic_ocr/ | For understanding OCR API call and response |
| Mistral Cloud Credentials: https://docs.n8n.io/integrations/builtin/credentials/mistral/ | Credential setup for OCR Document node |
| Slack OAuth2 Credentials: https://docs.n8n.io/integrations/builtin/credentials/slack/#using-oauth2 | Credential setup for Slack notifications |
| Google Sheets Configuration: Requires sheet named "Files" with specific headers: File Name, File, Directory, JSON File, MD File, Started, Completed, Status | To track processing jobs and statuses |
| Folder and Sheet IDs can be retrieved from their URLs in Google Drive and Sheets respectively | Important for proper configuration |
| Workflow expects support for PDF, DOCX, PNG, JPG, JPEG files; unsupported file types are logged and notified | Extend Mime type checks in Switch node to add more support |
| The workflow uses Mistral OCR API directly via HTTP Request node, not the built-in n8n Mistral Extract Text node, due to the need for image Base64 output | Important for output handling logic |
| Slack notifications include direct links to files and folders for easy access | Enhances monitoring and user awareness |
| Credentials must be configured before execution to avoid authentication errors | Critical for workflow operation |

---

This document provides a thorough understanding, detailed node-level analysis, and a complete step-by-step reproduction guide for the "Parse PDF, DOCX & Images with Mistral OCR via Google Drive with Slack Alerts" workflow. It enables advanced users and automation agents to maintain, extend, or integrate the workflow reliably.